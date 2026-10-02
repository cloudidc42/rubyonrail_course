# Part 89: Machine Learning ใน Ruby และ Rails

## บทนำ

Machine Learning (ML) กลายเป็นส่วนสำคัญของการพัฒนาซอฟต์แวร์ยุคใหม่ แม้ Python จะเป็นภาษาหลักสำหรับ ML แต่ Ruby ก็มีเครื่องมือที่ทรงพลังสำหรับงาน ML หลายประเภท บทนี้จะสอนการใช้ ML ใน Ruby และการ integrate เข้ากับ Rails applications

## 1. Machine Learning Basics ใน Ruby

### 1.1 ทำความเข้าใจ ML Concepts

```ruby
# Machine Learning แบ่งออกเป็น 3 ประเภทหลัก:
# 1. Supervised Learning - เรียนรู้จาก labeled data
# 2. Unsupervised Learning - หา pattern จาก unlabeled data
# 3. Reinforcement Learning - เรียนรู้จาก reward/punishment

# ตัวอย่าง Supervised Learning ง่ายๆ
class SimpleLinearRegression
  attr_reader :slope, :intercept
  
  def initialize
    @slope = 0.0
    @intercept = 0.0
  end
  
  # Train model ด้วย gradient descent
  def train(x_values, y_values, learning_rate: 0.01, epochs: 1000)
    n = x_values.length
    
    epochs.times do |epoch|
      # คำนวณ predictions
      predictions = x_values.map { |x| predict(x) }
      
      # คำนวณ errors
      errors = predictions.zip(y_values).map { |pred, actual| pred - actual }
      
      # Update slope and intercept
      slope_gradient = (2.0 / n) * x_values.zip(errors).sum { |x, e| x * e }
      intercept_gradient = (2.0 / n) * errors.sum
      
      @slope -= learning_rate * slope_gradient
      @intercept -= learning_rate * intercept_gradient
      
      # แสดง loss ทุก 100 epochs
      if epoch % 100 == 0
        loss = mean_squared_error(predictions, y_values)
        puts "Epoch #{epoch}: Loss = #{loss.round(4)}"
      end
    end
  end
  
  def predict(x)
    @slope * x + @intercept
  end
  
  private
  
  def mean_squared_error(predictions, actuals)
    n = predictions.length
    predictions.zip(actuals).sum { |pred, actual| (pred - actual) ** 2 } / n
  end
end

# ทดสอบ
regressor = SimpleLinearRegression.new
x_train = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
y_train = [2.1, 4.0, 5.9, 8.1, 10.0, 12.2, 14.1, 16.0, 17.9, 20.1]

regressor.train(x_train, y_train, learning_rate: 0.01, epochs: 1000)
puts "Slope: #{regressor.slope.round(4)}"
puts "Intercept: #{regressor.intercept.round(4)}"
puts "Prediction for x=11: #{regressor.predict(11).round(2)}"
```

### 1.2 Data Preprocessing

```ruby
# data_preprocessor.rb
class DataPreprocessor
  # Normalize data ให้อยู่ในช่วง [0, 1]
  def self.min_max_normalize(data)
    min_val = data.min
    max_val = data.max
    range = max_val - min_val
    
    return data.map { 0.5 } if range == 0  # กรณีทุก value เท่ากัน
    
    data.map { |x| (x - min_val).to_f / range }
  end
  
  # Standardize data (z-score normalization)
  def self.standardize(data)
    mean = data.sum.to_f / data.length
    variance = data.sum { |x| (x - mean) ** 2 } / data.length
    std_dev = Math.sqrt(variance)
    
    return data.map { 0.0 } if std_dev == 0
    
    data.map { |x| (x - mean) / std_dev }
  end
  
  # One-hot encoding สำหรับ categorical data
  def self.one_hot_encode(categories)
    unique_categories = categories.uniq.sort
    
    categories.map do |cat|
      unique_categories.map { |unique| cat == unique ? 1 : 0 }
    end
  end
  
  # Handle missing values
  def self.fill_missing(data, strategy: :mean)
    valid_values = data.compact
    
    fill_value = case strategy
    when :mean then valid_values.sum.to_f / valid_values.length
    when :median
      sorted = valid_values.sort
      n = sorted.length
      n.odd? ? sorted[n / 2] : (sorted[n / 2 - 1] + sorted[n / 2]) / 2.0
    when :mode then valid_values.tally.max_by { |_, count| count }[0]
    else 0
    end
    
    data.map { |x| x.nil? ? fill_value : x }
  end
  
  # Split data เป็น train/test sets
  def self.train_test_split(features, labels, test_size: 0.2, random_seed: 42)
    srand(random_seed)
    n = features.length
    indices = (0...n).to_a.shuffle
    
    test_count = (n * test_size).round
    test_indices = indices.first(test_count)
    train_indices = indices.last(n - test_count)
    
    {
      x_train: train_indices.map { |i| features[i] },
      x_test: test_indices.map { |i| features[i] },
      y_train: train_indices.map { |i| labels[i] },
      y_test: test_indices.map { |i| labels[i] }
    }
  end
end

# ทดสอบ
data = [1.0, 2.0, nil, 4.0, 5.0, nil, 7.0]
puts DataPreprocessor.fill_missing(data, strategy: :mean).inspect

categories = ['cat', 'dog', 'cat', 'bird', 'dog']
puts DataPreprocessor.one_hot_encode(categories).inspect
```

## 2. Numo::NArray สำหรับ Numerical Computing

### 2.1 การติดตั้งและใช้งาน Numo::NArray

```ruby
# Gemfile
gem 'numo-narray'
gem 'numo-linalg'  # สำหรับ linear algebra

# พื้นฐาน Numo::NArray
require 'numo/narray'

# สร้าง array
a = Numo::DFloat[1, 2, 3, 4, 5]
b = Numo::DFloat[2, 3, 4, 5, 6]

puts "Array a: #{a}"
puts "Array b: #{b}"
puts "a + b: #{a + b}"
puts "a * b: #{a * b}"  # element-wise multiplication
puts "dot product: #{a.dot(b)}"

# สร้าง 2D array (matrix)
matrix = Numo::DFloat[[1, 2, 3], [4, 5, 6], [7, 8, 9]]
puts "Matrix:\n#{matrix}"
puts "Shape: #{matrix.shape.inspect}"
puts "Transpose:\n#{matrix.transpose}"

# Operations
puts "Sum: #{matrix.sum}"
puts "Mean: #{matrix.mean}"
puts "Max: #{matrix.max}"
puts "Min: #{matrix.min}"

# Slicing
puts "First row: #{matrix[0, true]}"
puts "First column: #{matrix[true, 0]}"
puts "Sub-matrix:\n#{matrix[0..1, 0..1]}"
```

### 2.2 Matrix Operations และ Linear Algebra

```ruby
require 'numo/narray'
require 'numo/linalg'

# Matrix multiplication
a = Numo::DFloat[[1, 2], [3, 4]]
b = Numo::DFloat[[5, 6], [7, 8]]
c = a.dot(b)
puts "Matrix multiplication:\n#{c}"

# Eigenvalue decomposition
eigenvalues, eigenvectors = Numo::Linalg.eig(a)
puts "Eigenvalues: #{eigenvalues}"
puts "Eigenvectors:\n#{eigenvectors}"

# Singular Value Decomposition (SVD)
m = Numo::DFloat[[1, 2, 3], [4, 5, 6], [7, 8, 9]]
u, s, vt = Numo::Linalg.svd(m)
puts "SVD - U:\n#{u}"
puts "SVD - Singular values: #{s}"
puts "SVD - V^T:\n#{vt}"

# Solving linear equations Ax = b
# 2x + 3y = 8
# x + 4y = 7
a = Numo::DFloat[[2, 3], [1, 4]]
b = Numo::DFloat[8, 7]
x = Numo::Linalg.solve(a, b)
puts "Solution: x=#{x[0].round(4)}, y=#{x[1].round(4)}"

# หา inverse matrix
inv_a = Numo::Linalg.inv(a)
puts "Inverse of A:\n#{inv_a}"

# ตรวจสอบว่า A * A^-1 = I
identity = a.dot(inv_a)
puts "A * A^-1 ≈ I:\n#{identity.map { |x| x.round(6) }}"
```

### 2.3 Statistical Operations ด้วย Numo

```ruby
require 'numo/narray'

# สร้างข้อมูล sample
srand(42)
data = Numo::DFloat.new(1000).rand_norm(0, 1)  # Standard normal distribution

# Basic statistics
puts "Count: #{data.size}"
puts "Mean: #{data.mean.round(4)}"
puts "Std: #{data.std.round(4)}"
puts "Variance: #{data.var.round(4)}"
puts "Min: #{data.min.round(4)}"
puts "Max: #{data.max.round(4)}"

# Percentiles
sorted_data = data.sort
puts "25th percentile: #{sorted_data[250].round(4)}"
puts "50th percentile (median): #{sorted_data[500].round(4)}"
puts "75th percentile: #{sorted_data[750].round(4)}"

# Correlation matrix
x = Numo::DFloat.new(100).rand_norm(0, 1)
y = x * 0.8 + Numo::DFloat.new(100).rand_norm(0, 0.5)
z = Numo::DFloat.new(100).rand_norm(0, 1)

def correlation(a, b)
  n = a.size
  mean_a = a.mean
  mean_b = b.mean
  
  numerator = ((a - mean_a) * (b - mean_b)).sum
  denominator = Math.sqrt(((a - mean_a) ** 2).sum * ((b - mean_b) ** 2).sum)
  
  numerator / denominator
end

puts "\nCorrelation Matrix:"
puts "  X    Y    Z"
puts "X #{correlation(x, x).round(2)} #{correlation(x, y).round(2)} #{correlation(x, z).round(2)}"
puts "Y #{correlation(y, x).round(2)} #{correlation(y, y).round(2)} #{correlation(y, z).round(2)}"
puts "Z #{correlation(z, x).round(2)} #{correlation(z, y).round(2)} #{correlation(z, z).round(2)}"
```

## 3. Simple ML Algorithms ด้วย Pure Ruby

### 3.1 K-Nearest Neighbors (KNN)

```ruby
class KNearestNeighbors
  def initialize(k: 3)
    @k = k
    @training_data = []
    @training_labels = []
  end
  
  def fit(features, labels)
    @training_data = features
    @training_labels = labels
    self
  end
  
  def predict(features)
    features.map { |point| classify(point) }
  end
  
  def predict_proba(point)
    distances = @training_data.map.with_index do |train_point, idx|
      [euclidean_distance(point, train_point), @training_labels[idx]]
    end
    
    k_nearest = distances.sort_by { |d, _| d }.first(@k)
    label_counts = k_nearest.map { |_, label| label }.tally
    total = k_nearest.length.to_f
    
    label_counts.transform_values { |count| count / total }
  end
  
  private
  
  def classify(point)
    distances = @training_data.map.with_index do |train_point, idx|
      [euclidean_distance(point, train_point), @training_labels[idx]]
    end
    
    k_nearest = distances.sort_by { |d, _| d }.first(@k)
    k_nearest.map { |_, label| label }.tally.max_by { |_, count| count }[0]
  end
  
  def euclidean_distance(a, b)
    Math.sqrt(a.zip(b).sum { |x, y| (x - y) ** 2 })
  end
end

# ทดสอบ KNN สำหรับ Iris classification
training_data = [
  [5.1, 3.5, 1.4, 0.2],
  [4.9, 3.0, 1.4, 0.2],
  [7.0, 3.2, 4.7, 1.4],
  [6.4, 3.2, 4.5, 1.5],
  [6.3, 3.3, 6.0, 2.5],
  [5.8, 2.7, 5.1, 1.9]
]
training_labels = ['setosa', 'setosa', 'versicolor', 'versicolor', 'virginica', 'virginica']

knn = KNearestNeighbors.new(k: 3)
knn.fit(training_data, training_labels)

test_point = [5.0, 3.4, 1.5, 0.2]
prediction = knn.predict([test_point])
puts "Prediction: #{prediction[0]}"
puts "Probabilities: #{knn.predict_proba(test_point)}"
```

### 3.2 Decision Tree

```ruby
class DecisionTree
  attr_accessor :max_depth, :min_samples_split
  
  def initialize(max_depth: 5, min_samples_split: 2)
    @max_depth = max_depth
    @min_samples_split = min_samples_split
    @tree = nil
  end
  
  def fit(features, labels)
    @tree = build_tree(features, labels, depth: 0)
    self
  end
  
  def predict(features)
    features.map { |point| traverse_tree(@tree, point) }
  end
  
  private
  
  def build_tree(features, labels, depth:)
    return { type: :leaf, label: majority_label(labels) } if stop_condition?(labels, depth)
    
    best_split = find_best_split(features, labels)
    return { type: :leaf, label: majority_label(labels) } if best_split.nil?
    
    left_features, left_labels, right_features, right_labels = split_data(
      features, labels, best_split[:feature_idx], best_split[:threshold]
    )
    
    {
      type: :node,
      feature_idx: best_split[:feature_idx],
      threshold: best_split[:threshold],
      gini: best_split[:gini],
      left: build_tree(left_features, left_labels, depth: depth + 1),
      right: build_tree(right_features, right_labels, depth: depth + 1)
    }
  end
  
  def find_best_split(features, labels)
    best_gini = Float::INFINITY
    best_split = nil
    
    n_features = features[0].length
    
    n_features.times do |feature_idx|
      thresholds = features.map { |f| f[feature_idx] }.uniq.sort
      
      thresholds.each do |threshold|
        left_indices = features.each_index.select { |i| features[i][feature_idx] <= threshold }
        right_indices = features.each_index.select { |i| features[i][feature_idx] > threshold }
        
        next if left_indices.empty? || right_indices.empty?
        
        left_labels = left_indices.map { |i| labels[i] }
        right_labels = right_indices.map { |i| labels[i] }
        
        gini = weighted_gini(left_labels, right_labels)
        
        if gini < best_gini
          best_gini = gini
          best_split = { feature_idx: feature_idx, threshold: threshold, gini: gini }
        end
      end
    end
    
    best_split
  end
  
  def gini_impurity(labels)
    return 0 if labels.empty?
    n = labels.length.to_f
    label_counts = labels.tally
    1 - label_counts.values.sum { |count| (count / n) ** 2 }
  end
  
  def weighted_gini(left_labels, right_labels)
    total = (left_labels.length + right_labels.length).to_f
    (left_labels.length / total) * gini_impurity(left_labels) +
      (right_labels.length / total) * gini_impurity(right_labels)
  end
  
  def split_data(features, labels, feature_idx, threshold)
    left_features, left_labels = [], []
    right_features, right_labels = [], []
    
    features.each_with_index do |feature, i|
      if feature[feature_idx] <= threshold
        left_features << feature
        left_labels << labels[i]
      else
        right_features << feature
        right_labels << labels[i]
      end
    end
    
    [left_features, left_labels, right_features, right_labels]
  end
  
  def traverse_tree(node, point)
    return node[:label] if node[:type] == :leaf
    
    if point[node[:feature_idx]] <= node[:threshold]
      traverse_tree(node[:left], point)
    else
      traverse_tree(node[:right], point)
    end
  end
  
  def majority_label(labels)
    labels.tally.max_by { |_, count| count }[0]
  end
  
  def stop_condition?(labels, depth)
    labels.uniq.length == 1 ||
      depth >= @max_depth ||
      labels.length < @min_samples_split
  end
end

# ทดสอบ Decision Tree
features = [
  [2.7, 2.5], [1.4, 2.3], [3.3, 4.4], [1.3, 1.8], [3.0, 3.0],
  [7.6, 2.7], [5.3, 2.0], [6.9, 1.7], [2.6, 1.2], [5.0, 3.3]
]
labels = [0, 0, 0, 0, 0, 1, 1, 1, 1, 1]

tree = DecisionTree.new(max_depth: 3)
tree.fit(features, labels)
predictions = tree.predict([[3.0, 3.0], [6.0, 2.5]])
puts "Predictions: #{predictions.inspect}"
```

### 3.3 K-Means Clustering

```ruby
class KMeans
  attr_reader :centroids, :labels
  
  def initialize(k:, max_iterations: 300, tolerance: 1e-4, random_seed: 42)
    @k = k
    @max_iterations = max_iterations
    @tolerance = tolerance
    @random_seed = random_seed
  end
  
  def fit(data)
    srand(@random_seed)
    @centroids = initialize_centroids(data)
    
    @max_iterations.times do |iteration|
      old_centroids = @centroids.map(&:dup)
      
      # Assign labels
      @labels = data.map { |point| nearest_centroid(point) }
      
      # Update centroids
      @k.times do |k|
        cluster_points = data.select.with_index { |_, i| @labels[i] == k }
        
        unless cluster_points.empty?
          @centroids[k] = cluster_mean(cluster_points)
        end
      end
      
      # Check convergence
      shift = old_centroids.zip(@centroids).sum do |old, new|
        euclidean_distance(old, new)
      end
      
      puts "Iteration #{iteration + 1}: centroid shift = #{shift.round(6)}"
      break if shift < @tolerance
    end
    
    self
  end
  
  def predict(data)
    data.map { |point| nearest_centroid(point) }
  end
  
  def inertia(data)
    @labels ||= predict(data)
    data.each_with_index.sum do |point, i|
      euclidean_distance(point, @centroids[@labels[i]]) ** 2
    end
  end
  
  private
  
  def initialize_centroids(data)
    data.sample(@k)
  end
  
  def nearest_centroid(point)
    @centroids.each_with_index.min_by do |centroid, _|
      euclidean_distance(point, centroid)
    end[1]
  end
  
  def euclidean_distance(a, b)
    Math.sqrt(a.zip(b).sum { |x, y| (x - y) ** 2 })
  end
  
  def cluster_mean(points)
    n = points.length.to_f
    dim = points[0].length
    
    dim.times.map do |d|
      points.sum { |p| p[d] } / n
    end
  end
end

# ทดสอบ K-Means
data = [
  [1.0, 2.0], [1.5, 1.8], [5.0, 8.0], [8.0, 8.0],
  [1.0, 0.6], [9.0, 11.0], [8.0, 2.0], [10.0, 2.0],
  [9.0, 3.0], [1.2, 2.5]
]

kmeans = KMeans.new(k: 3, max_iterations: 100)
kmeans.fit(data)
puts "\nCluster labels: #{kmeans.labels.inspect}"
puts "Centroids:"
kmeans.centroids.each_with_index { |c, i| puts "  Cluster #{i}: #{c.map { |x| x.round(2) }.inspect}" }
puts "Inertia: #{kmeans.inertia(data).round(2)}"
```

## 4. Ruby-Tensorflow Integration

### 4.1 การใช้ tensorflow.rb

```ruby
# Gemfile
gem 'tensorflow'

# การใช้ TensorFlow ใน Ruby
require 'tensorflow'

# สร้าง simple neural network
model = Tensorflow::Keras::Sequential.new

# เพิ่ม layers
model.add(Tensorflow::Keras::Layers::Dense.new(
  units: 64,
  activation: 'relu',
  input_shape: [10]  # 10 input features
))

model.add(Tensorflow::Keras::Layers::Dropout.new(rate: 0.2))

model.add(Tensorflow::Keras::Layers::Dense.new(
  units: 32,
  activation: 'relu'
))

model.add(Tensorflow::Keras::Layers::Dense.new(
  units: 1,
  activation: 'sigmoid'  # สำหรับ binary classification
))

# Compile model
model.compile(
  optimizer: 'adam',
  loss: 'binary_crossentropy',
  metrics: ['accuracy']
)

model.summary
```

### 4.2 Neural Network ด้วย Ruby (Pure Implementation)

```ruby
# neural_network.rb - Simple feedforward neural network
class NeuralNetwork
  def initialize(layers:, learning_rate: 0.01)
    @layers = layers
    @learning_rate = learning_rate
    @weights = []
    @biases = []
    
    initialize_parameters
  end
  
  def train(x_train, y_train, epochs: 100, batch_size: 32)
    n = x_train.length
    
    epochs.times do |epoch|
      # Mini-batch gradient descent
      indices = (0...n).to_a.shuffle
      
      total_loss = 0.0
      batches = indices.each_slice(batch_size).to_a
      
      batches.each do |batch_indices|
        x_batch = batch_indices.map { |i| x_train[i] }
        y_batch = batch_indices.map { |i| y_train[i] }
        
        # Forward pass
        activations, z_values = forward_pass(x_batch)
        
        # Compute loss
        predictions = activations.last
        loss = binary_crossentropy(predictions, y_batch)
        total_loss += loss * batch_indices.length
        
        # Backward pass
        gradients = backward_pass(x_batch, y_batch, activations, z_values)
        
        # Update parameters
        update_parameters(gradients)
      end
      
      avg_loss = total_loss / n
      puts "Epoch #{epoch + 1}/#{epochs}: Loss = #{avg_loss.round(4)}" if (epoch + 1) % 10 == 0
    end
  end
  
  def predict(x)
    activations, _ = forward_pass([x])
    activations.last[0]
  end
  
  private
  
  def initialize_parameters
    @layers.each_cons(2) do |in_size, out_size|
      # Xavier initialization
      scale = Math.sqrt(2.0 / (in_size + out_size))
      weights = Array.new(out_size) do
        Array.new(in_size) { (rand * 2 - 1) * scale }
      end
      @weights << weights
      @biases << Array.new(out_size, 0.0)
    end
  end
  
  def forward_pass(x_batch)
    activations = [x_batch]
    z_values = []
    
    @weights.each_with_index do |w, layer_idx|
      z_batch = x_batch.map do |x|
        w.map.with_index do |w_row, j|
          w_row.zip(x).sum { |wi, xi| wi * xi } + @biases[layer_idx][j]
        end
      end
      z_values << z_batch
      
      # Apply activation function
      if layer_idx == @weights.length - 1
        x_batch = z_batch.map { |z| z.map { |zi| sigmoid(zi) } }
      else
        x_batch = z_batch.map { |z| z.map { |zi| relu(zi) } }
      end
      
      activations << x_batch
    end
    
    [activations, z_values]
  end
  
  def backward_pass(x_batch, y_batch, activations, z_values)
    n = x_batch.length
    gradients = { weights: [], biases: [] }
    
    # Output layer delta
    delta = activations.last.zip(y_batch).map do |pred_batch, y_batch_item|
      pred_batch.zip([y_batch_item].flatten).map { |pred, y| pred - y }
    end
    
    # Backpropagate
    (@weights.length - 1).downto(0) do |layer_idx|
      # Compute gradients
      w_grad = @weights[layer_idx].map.with_index do |_, j|
        @weights[layer_idx][j].map.with_index do |_, k|
          delta.zip(activations[layer_idx]).sum do |d, a|
            d[j] * a[k]
          end / n
        end
      end
      
      b_grad = @biases[layer_idx].map.with_index do |_, j|
        delta.sum { |d| d[j] } / n
      end
      
      gradients[:weights].unshift(w_grad)
      gradients[:biases].unshift(b_grad)
      
      # Compute delta for previous layer
      if layer_idx > 0
        delta = activations[layer_idx].map.with_index do |a, batch_idx|
          a.map.with_index do |_, k|
            @weights[layer_idx].sum.with_index do |w_row, j|
              delta[batch_idx][j] * w_row[k] * relu_derivative(z_values[layer_idx - 1][batch_idx][k])
            end
          end
        end
      end
    end
    
    gradients
  end
  
  def update_parameters(gradients)
    @weights.each_with_index do |w, layer_idx|
      w.each_with_index do |w_row, j|
        w_row.each_with_index do |_, k|
          @weights[layer_idx][j][k] -= @learning_rate * gradients[:weights][layer_idx][j][k]
        end
      end
      
      @biases[layer_idx].each_with_index do |_, j|
        @biases[layer_idx][j] -= @learning_rate * gradients[:biases][layer_idx][j]
      end
    end
  end
  
  def sigmoid(x)
    1.0 / (1.0 + Math.exp(-x.clamp(-500, 500)))
  end
  
  def relu(x)
    [x, 0].max
  end
  
  def relu_derivative(x)
    x > 0 ? 1.0 : 0.0
  end
  
  def binary_crossentropy(predictions, actuals)
    eps = 1e-15
    predictions.zip(actuals).sum do |pred, actual|
      y = actual.is_a?(Array) ? actual[0] : actual
      p = pred[0].clamp(eps, 1 - eps)
      -(y * Math.log(p) + (1 - y) * Math.log(1 - p))
    end / predictions.length
  end
end

# ทดสอบ XOR problem
x_train = [[0, 0], [0, 1], [1, 0], [1, 1]]
y_train = [0, 1, 1, 0]

nn = NeuralNetwork.new(layers: [2, 4, 1], learning_rate: 0.1)
nn.train(x_train, y_train, epochs: 1000, batch_size: 4)

puts "\nXOR Predictions:"
x_train.zip(y_train).each do |x, y|
  pred = nn.predict(x)
  puts "  #{x.inspect} => #{pred.round(4)} (actual: #{y})"
end
```

## 5. Recommendation System ใน Rails (Collaborative Filtering)

### 5.1 User-Based Collaborative Filtering

```ruby
# app/services/recommendation_service.rb
class RecommendationService
  def initialize(user_id)
    @user = User.find(user_id)
  end
  
  # หา items ที่ควร recommend ให้ user
  def recommend(n_recommendations: 10)
    similar_users = find_similar_users
    candidate_items = collect_candidate_items(similar_users)
    
    scored_items = candidate_items.map do |item_id, user_ratings|
      score = calculate_predicted_rating(item_id, user_ratings, similar_users)
      { item_id: item_id, predicted_rating: score }
    end
    
    scored_items
      .sort_by { |item| -item[:predicted_rating] }
      .first(n_recommendations)
  end
  
  private
  
  def find_similar_users(n_users: 20)
    all_users = User.where.not(id: @user.id)
    
    user_ratings = get_user_ratings(@user)
    
    similarities = all_users.map do |other_user|
      other_ratings = get_user_ratings(other_user)
      similarity = pearson_correlation(user_ratings, other_ratings)
      
      { user: other_user, similarity: similarity }
    end
    
    similarities
      .select { |s| s[:similarity] > 0 }
      .sort_by { |s| -s[:similarity] }
      .first(n_users)
  end
  
  def get_user_ratings(user)
    user.ratings.includes(:item).each_with_object({}) do |rating, hash|
      hash[rating.item_id] = rating.score
    end
  end
  
  def pearson_correlation(ratings_a, ratings_b)
    common_items = ratings_a.keys & ratings_b.keys
    return 0.0 if common_items.length < 2
    
    a_values = common_items.map { |id| ratings_a[id].to_f }
    b_values = common_items.map { |id| ratings_b[id].to_f }
    
    n = common_items.length.to_f
    
    mean_a = a_values.sum / n
    mean_b = b_values.sum / n
    
    numerator = a_values.zip(b_values).sum do |a, b|
      (a - mean_a) * (b - mean_b)
    end
    
    std_a = Math.sqrt(a_values.sum { |a| (a - mean_a) ** 2 })
    std_b = Math.sqrt(b_values.sum { |b| (b - mean_b) ** 2 })
    
    return 0.0 if std_a == 0 || std_b == 0
    
    numerator / (std_a * std_b)
  end
  
  def collect_candidate_items(similar_users)
    user_rated_items = get_user_ratings(@user).keys
    
    candidates = {}
    
    similar_users.each do |similar|
      get_user_ratings(similar[:user]).each do |item_id, rating|
        next if user_rated_items.include?(item_id)
        
        candidates[item_id] ||= []
        candidates[item_id] << {
          rating: rating,
          similarity: similar[:similarity],
          user: similar[:user]
        }
      end
    end
    
    candidates
  end
  
  def calculate_predicted_rating(item_id, user_ratings, similar_users)
    user_avg = calculate_average_rating(@user)
    
    numerator = user_ratings.sum do |rating_info|
      user = rating_info[:user]
      similarity = rating_info[:similarity]
      rating = rating_info[:rating]
      other_avg = calculate_average_rating(user)
      
      similarity * (rating - other_avg)
    end
    
    denominator = user_ratings.sum { |r| r[:similarity].abs }
    
    return user_avg if denominator == 0
    
    user_avg + (numerator / denominator)
  end
  
  def calculate_average_rating(user)
    ratings = user.ratings.pluck(:score)
    return 0.0 if ratings.empty?
    
    ratings.sum.to_f / ratings.length
  end
end
```

### 5.2 Item-Based Collaborative Filtering

```ruby
# app/services/item_similarity_service.rb
class ItemSimilarityService
  include Rails.application.routes.url_helpers
  
  SIMILARITY_CACHE_KEY = "item_similarities"
  CACHE_EXPIRY = 1.hour
  
  # สร้าง item-item similarity matrix
  def self.build_similarity_matrix
    items = Item.all.to_a
    matrix = {}
    
    items.combination(2).each do |item_a, item_b|
      similarity = cosine_similarity(item_a, item_b)
      
      matrix[item_a.id] ||= {}
      matrix[item_b.id] ||= {}
      
      matrix[item_a.id][item_b.id] = similarity
      matrix[item_b.id][item_a.id] = similarity
    end
    
    Rails.cache.write(SIMILARITY_CACHE_KEY, matrix, expires_in: CACHE_EXPIRY)
    matrix
  end
  
  def self.similar_items(item_id, n: 10)
    matrix = Rails.cache.fetch(SIMILARITY_CACHE_KEY) { build_similarity_matrix }
    
    similarities = matrix[item_id] || {}
    
    similarities
      .sort_by { |_, sim| -sim }
      .first(n)
      .map { |id, sim| { item: Item.find(id), similarity: sim } }
  end
  
  private
  
  def self.item_vector(item)
    # แปลง item ratings เป็น vector
    all_users = User.pluck(:id)
    
    all_users.map do |user_id|
      rating = Rating.find_by(item: item, user_id: user_id)
      rating&.score || 0.0
    end
  end
  
  def self.cosine_similarity(item_a, item_b)
    vec_a = item_vector(item_a)
    vec_b = item_vector(item_b)
    
    dot_product = vec_a.zip(vec_b).sum { |a, b| a * b }
    
    magnitude_a = Math.sqrt(vec_a.sum { |x| x ** 2 })
    magnitude_b = Math.sqrt(vec_b.sum { |x| x ** 2 })
    
    return 0.0 if magnitude_a == 0 || magnitude_b == 0
    
    dot_product / (magnitude_a * magnitude_b)
  end
end

# Controller สำหรับ recommendations
class RecommendationsController < ApplicationController
  before_action :authenticate_user!
  
  def index
    @recommendations = RecommendationService.new(current_user.id)
      .recommend(n_recommendations: 20)
    
    @items = Item.where(
      id: @recommendations.map { |r| r[:item_id] }
    ).index_by(&:id)
    
    render json: @recommendations.map do |rec|
      item = @items[rec[:item_id]]
      {
        item: ItemSerializer.new(item).as_json,
        predicted_rating: rec[:predicted_rating].round(2)
      }
    end
  end
  
  def similar
    item = Item.find(params[:item_id])
    similar = ItemSimilarityService.similar_items(item.id, n: 10)
    
    render json: similar.map do |s|
      {
        item: ItemSerializer.new(s[:item]).as_json,
        similarity: s[:similarity].round(4)
      }
    end
  end
end
```

### 5.3 Matrix Factorization (SVD-based)

```ruby
# app/services/matrix_factorization_service.rb
class MatrixFactorizationService
  def initialize(n_factors: 20, learning_rate: 0.005, regularization: 0.02, epochs: 100)
    @n_factors = n_factors
    @learning_rate = learning_rate
    @regularization = regularization
    @epochs = epochs
  end
  
  def train
    @user_ids = User.pluck(:id)
    @item_ids = Item.pluck(:id)
    
    n_users = @user_ids.length
    n_items = @item_ids.length
    
    # Initialize factor matrices randomly
    @user_factors = Array.new(n_users) { Array.new(@n_factors) { rand * 0.1 } }
    @item_factors = Array.new(n_items) { Array.new(@n_factors) { rand * 0.1 } }
    @user_biases = Array.new(n_users, 0.0)
    @item_biases = Array.new(n_items, 0.0)
    
    # Global bias
    ratings = Rating.pluck(:score)
    @global_bias = ratings.sum.to_f / ratings.length
    
    # User/item index mapping
    @user_index = @user_ids.each_with_index.to_h
    @item_index = @item_ids.each_with_index.to_h
    
    @epochs.times do |epoch|
      total_error = 0.0
      
      Rating.find_each do |rating|
        u_idx = @user_index[rating.user_id]
        i_idx = @item_index[rating.item_id]
        
        next if u_idx.nil? || i_idx.nil?
        
        # Predicted rating
        prediction = predict_rating(u_idx, i_idx)
        error = rating.score - prediction
        total_error += error ** 2
        
        # Update biases
        @user_biases[u_idx] += @learning_rate * (error - @regularization * @user_biases[u_idx])
        @item_biases[i_idx] += @learning_rate * (error - @regularization * @item_biases[i_idx])
        
        # Update factor matrices
        user_factor = @user_factors[u_idx].dup
        item_factor = @item_factors[i_idx].dup
        
        @n_factors.times do |f|
          @user_factors[u_idx][f] += @learning_rate * (
            error * item_factor[f] - @regularization * user_factor[f]
          )
          
          @item_factors[i_idx][f] += @learning_rate * (
            error * user_factor[f] - @regularization * item_factor[f]
          )
        end
      end
      
      rmse = Math.sqrt(total_error / Rating.count)
      puts "Epoch #{epoch + 1}: RMSE = #{rmse.round(4)}" if (epoch + 1) % 10 == 0
    end
    
    save_model
    self
  end
  
  def recommend_for_user(user_id, n: 10)
    u_idx = @user_index[user_id]
    return [] if u_idx.nil?
    
    rated_item_ids = Rating.where(user_id: user_id).pluck(:item_id)
    
    predictions = @item_ids.each_with_index.map do |item_id, i_idx|
      next if rated_item_ids.include?(item_id)
      
      {
        item_id: item_id,
        predicted_rating: predict_rating(u_idx, i_idx)
      }
    end.compact
    
    predictions.sort_by { |p| -p[:predicted_rating] }.first(n)
  end
  
  private
  
  def predict_rating(u_idx, i_idx)
    dot_product = @user_factors[u_idx].zip(@item_factors[i_idx])
      .sum { |u, v| u * v }
    
    (@global_bias + @user_biases[u_idx] + @item_biases[i_idx] + dot_product)
      .clamp(1.0, 5.0)
  end
  
  def save_model
    model_data = {
      user_ids: @user_ids,
      item_ids: @item_ids,
      user_factors: @user_factors,
      item_factors: @item_factors,
      user_biases: @user_biases,
      item_biases: @item_biases,
      global_bias: @global_bias
    }
    
    Rails.cache.write('mf_model', model_data, expires_in: 24.hours)
  end
end
```

## 6. Text Classification ด้วย Naive Bayes

### 6.1 Naive Bayes Classifier

```ruby
# app/services/naive_bayes_classifier.rb
class NaiveBayesClassifier
  def initialize
    @class_counts = Hash.new(0)
    @word_counts = Hash.new { |h, k| h[k] = Hash.new(0) }
    @vocabulary = Set.new
    @total_documents = 0
  end
  
  def train(text, label)
    @total_documents += 1
    @class_counts[label] += 1
    
    tokenize(text).each do |word|
      @word_counts[label][word] += 1
      @vocabulary.add(word)
    end
    
    self
  end
  
  def train_batch(training_data)
    training_data.each do |item|
      train(item[:text], item[:label])
    end
    self
  end
  
  def predict(text)
    scores = calculate_scores(text)
    scores.max_by { |_, score| score }[0]
  end
  
  def predict_proba(text)
    scores = calculate_scores(text)
    
    # Convert log probabilities to probabilities
    max_score = scores.values.max
    exp_scores = scores.transform_values { |s| Math.exp(s - max_score) }
    total = exp_scores.values.sum
    
    exp_scores.transform_values { |s| s / total }
  end
  
  def accuracy(test_data)
    correct = test_data.count { |item| predict(item[:text]) == item[:label] }
    correct.to_f / test_data.length
  end
  
  def confusion_matrix(test_data)
    labels = @class_counts.keys
    matrix = labels.each_with_object({}) do |label, h|
      h[label] = labels.each_with_object({}) { |l2, h2| h2[l2] = 0 }
    end
    
    test_data.each do |item|
      actual = item[:label]
      predicted = predict(item[:text])
      matrix[actual][predicted] += 1
    end
    
    matrix
  end
  
  private
  
  def tokenize(text)
    # Simple tokenization - ลบ stop words และ normalize
    stop_words = %w[the a an is are was were be been being have has had
                    do does did will would could should may might must
                    shall can need dare ought used to]
    
    text.downcase
        .gsub(/[^a-z\s]/, ' ')
        .split
        .reject { |word| stop_words.include?(word) || word.length < 2 }
  end
  
  def calculate_scores(text)
    words = tokenize(text)
    vocab_size = @vocabulary.size
    
    @class_counts.each_with_object({}) do |(label, count), scores|
      # Prior probability (log)
      prior = Math.log(count.to_f / @total_documents)
      
      # Likelihood (log) with Laplace smoothing
      total_words = @word_counts[label].values.sum + vocab_size
      
      likelihood = words.sum do |word|
        word_count = @word_counts[label][word] + 1  # Add-1 smoothing
        Math.log(word_count.to_f / total_words)
      end
      
      scores[label] = prior + likelihood
    end
  end
end

# ทดสอบ Text Classification
classifier = NaiveBayesClassifier.new

# Training data (sentiment analysis)
training_data = [
  { text: "This movie was amazing and wonderful", label: "positive" },
  { text: "Great film with excellent acting", label: "positive" },
  { text: "I loved every minute of this movie", label: "positive" },
  { text: "Fantastic story and beautiful cinematography", label: "positive" },
  { text: "Terrible movie with bad acting", label: "negative" },
  { text: "Boring and disappointing film", label: "negative" },
  { text: "Waste of time and money", label: "negative" },
  { text: "Awful plot with poor dialogue", label: "negative" }
]

classifier.train_batch(training_data)

test_texts = [
  "Wonderful and amazing performance",
  "Really bad and boring experience",
  "Great acting but terrible plot"
]

test_texts.each do |text|
  prediction = classifier.predict(text)
  proba = classifier.predict_proba(text)
  puts "Text: '#{text}'"
  puts "Prediction: #{prediction}"
  puts "Probabilities: #{proba.transform_values { |v| v.round(4) }}"
  puts
end
```

### 6.2 Rails Integration สำหรับ Text Classification

```ruby
# app/models/spam_detector.rb
class SpamDetector
  MODEL_CACHE_KEY = 'spam_detector_model'
  
  def self.instance
    @instance ||= new
  end
  
  def initialize
    @classifier = NaiveBayesClassifier.new
    load_or_train_model
  end
  
  def spam?(text)
    predict(text) == 'spam'
  end
  
  def classify(text)
    {
      label: predict(text),
      confidence: predict_proba(text)
    }
  end
  
  def retrain!
    @classifier = NaiveBayesClassifier.new
    
    # Load training data from database
    SpamTrainingData.find_each do |data|
      @classifier.train(data.text, data.label)
    end
    
    Rails.cache.write(MODEL_CACHE_KEY, @classifier)
    Rails.logger.info "Spam detector retrained with #{SpamTrainingData.count} samples"
  end
  
  private
  
  def predict(text)
    @classifier.predict(text)
  end
  
  def predict_proba(text)
    @classifier.predict_proba(text)
  end
  
  def load_or_train_model
    if (cached_model = Rails.cache.read(MODEL_CACHE_KEY))
      @classifier = cached_model
    else
      retrain!
    end
  end
end

# app/controllers/comments_controller.rb
class CommentsController < ApplicationController
  def create
    @comment = Comment.new(comment_params)
    @comment.user = current_user
    
    # ตรวจสอบ spam
    spam_result = SpamDetector.instance.classify(@comment.body)
    
    if spam_result[:label] == 'spam' && spam_result[:confidence]['spam'] > 0.9
      @comment.mark_as_spam!
      render json: { error: 'Comment flagged as spam' }, status: :unprocessable_entity
      return
    end
    
    if @comment.save
      render json: CommentSerializer.new(@comment), status: :created
    else
      render json: { errors: @comment.errors }, status: :unprocessable_entity
    end
  end
  
  private
  
  def comment_params
    params.require(:comment).permit(:body, :post_id)
  end
end
```

### 6.3 TF-IDF Feature Extraction

```ruby
class TfIdfVectorizer
  def initialize(max_features: 1000, min_df: 2, max_df: 0.95)
    @max_features = max_features
    @min_df = min_df
    @max_df = max_df
    @vocabulary = {}
    @idf = {}
  end
  
  def fit(documents)
    n_docs = documents.length
    
    # Count document frequency for each term
    df = Hash.new(0)
    documents.each do |doc|
      tokenize(doc).uniq.each { |term| df[term] += 1 }
    end
    
    # Filter terms by document frequency
    valid_terms = df.select do |term, count|
      count >= @min_df && (count.to_f / n_docs) <= @max_df
    end
    
    # Sort by frequency and take top max_features
    top_terms = valid_terms
      .sort_by { |_, count| -count }
      .first(@max_features)
      .map { |term, _| term }
    
    # Build vocabulary
    @vocabulary = top_terms.each_with_index.to_h
    
    # Calculate IDF
    @idf = @vocabulary.keys.each_with_object({}) do |term, idf|
      idf[term] = Math.log((n_docs + 1).to_f / (df[term] + 1)) + 1
    end
    
    self
  end
  
  def transform(documents)
    documents.map do |doc|
      vector = Array.new(@vocabulary.size, 0.0)
      
      term_counts = tokenize(doc).tally
      total_terms = term_counts.values.sum.to_f
      
      term_counts.each do |term, count|
        next unless @vocabulary.key?(term)
        
        tf = count / total_terms  # Term frequency
        tfidf = tf * @idf[term]   # TF-IDF score
        
        vector[@vocabulary[term]] = tfidf
      end
      
      # L2 normalize
      magnitude = Math.sqrt(vector.sum { |v| v ** 2 })
      magnitude > 0 ? vector.map { |v| v / magnitude } : vector
    end
  end
  
  def fit_transform(documents)
    fit(documents).transform(documents)
  end
  
  private
  
  def tokenize(text)
    text.downcase.gsub(/[^a-z\s]/, ' ').split.select { |w| w.length > 2 }
  end
end

# ใช้งาน TF-IDF
documents = [
  "Ruby is a great programming language",
  "Rails makes web development easy with Ruby",
  "Machine learning with Python and Ruby",
  "TensorFlow for deep learning applications"
]

vectorizer = TfIdfVectorizer.new(max_features: 50)
vectors = vectorizer.fit_transform(documents)

puts "TF-IDF Vectors shape: #{vectors.length} documents × #{vectors[0].length} features"
puts "First document vector (non-zero):"
vectors[0].each_with_index do |val, idx|
  puts "  Feature #{idx}: #{val.round(4)}" if val > 0
end
```

## 7. Background Jobs สำหรับ ML Training

```ruby
# app/jobs/train_recommendation_model_job.rb
class TrainRecommendationModelJob < ApplicationJob
  queue_as :ml_training
  
  def perform
    Rails.logger.info "Starting recommendation model training..."
    start_time = Time.current
    
    begin
      service = MatrixFactorizationService.new(
        n_factors: 50,
        learning_rate: 0.005,
        regularization: 0.02,
        epochs: 50
      )
      
      service.train
      
      duration = Time.current - start_time
      Rails.logger.info "Model training completed in #{duration.round(2)}s"
      
      # Notify admin
      AdminMailer.model_trained_notification(duration: duration).deliver_later
    rescue => e
      Rails.logger.error "Model training failed: #{e.message}"
      raise
    end
  end
end

# app/jobs/generate_recommendations_job.rb
class GenerateRecommendationsJob < ApplicationJob
  queue_as :recommendations
  
  def perform(user_id)
    user = User.find(user_id)
    service = RecommendationService.new(user_id)
    
    recommendations = service.recommend(n_recommendations: 50)
    
    # Store recommendations in cache
    cache_key = "recommendations:user:#{user_id}"
    Rails.cache.write(cache_key, recommendations, expires_in: 6.hours)
    
    # Optionally, store in database for persistence
    user.recommendations.delete_all
    
    recommendations.each do |rec|
      user.recommendations.create!(
        item_id: rec[:item_id],
        predicted_rating: rec[:predicted_rating],
        generated_at: Time.current
      )
    end
    
    Rails.logger.info "Generated #{recommendations.length} recommendations for user #{user_id}"
  end
end
```

## 8. A/B Testing สำหรับ ML Models

```ruby
# app/services/ab_test_service.rb
class AbTestService
  EXPERIMENTS = {
    'recommendation_algorithm' => {
      variants: {
        'collaborative_filtering' => 0.5,
        'matrix_factorization' => 0.3,
        'content_based' => 0.2
      }
    }
  }
  
  def self.assign_variant(user_id, experiment_name)
    experiment = EXPERIMENTS[experiment_name]
    return nil unless experiment
    
    # Deterministic assignment based on user_id
    hash = Digest::MD5.hexdigest("#{user_id}:#{experiment_name}").to_i(16)
    value = (hash % 100) / 100.0
    
    cumulative = 0.0
    experiment[:variants].each do |variant, probability|
      cumulative += probability
      return variant if value < cumulative
    end
    
    experiment[:variants].keys.last
  end
  
  def self.track_event(user_id, experiment_name, event, value: 1)
    variant = assign_variant(user_id, experiment_name)
    
    AbTestEvent.create!(
      user_id: user_id,
      experiment: experiment_name,
      variant: variant,
      event: event,
      value: value
    )
  end
  
  def self.analyze_results(experiment_name)
    events = AbTestEvent.where(experiment: experiment_name)
    
    results = events.group(:variant).group(:event).sum(:value)
    
    variants = events.group(:variant).count
    
    {
      total_users: variants,
      events_by_variant: results,
      conversion_rates: calculate_conversion_rates(events)
    }
  end
  
  private
  
  def self.calculate_conversion_rates(events)
    total_by_variant = events.group(:variant).count
    
    conversions_by_variant = events
      .where(event: 'click')
      .group(:variant)
      .count
    
    total_by_variant.each_with_object({}) do |(variant, total), rates|
      clicks = conversions_by_variant[variant] || 0
      rates[variant] = total > 0 ? (clicks.to_f / total * 100).round(2) : 0
    end
  end
end
```

## 9. Model Evaluation และ Metrics

```ruby
# lib/ml/evaluation.rb
module ML
  module Evaluation
    # Classification metrics
    def self.accuracy(y_true, y_pred)
      correct = y_true.zip(y_pred).count { |true_val, pred_val| true_val == pred_val }
      correct.to_f / y_true.length
    end
    
    def self.precision(y_true, y_pred, positive_class: 1)
      tp = y_true.zip(y_pred).count { |t, p| t == positive_class && p == positive_class }
      fp = y_true.zip(y_pred).count { |t, p| t != positive_class && p == positive_class }
      
      tp + fp == 0 ? 0.0 : tp.to_f / (tp + fp)
    end
    
    def self.recall(y_true, y_pred, positive_class: 1)
      tp = y_true.zip(y_pred).count { |t, p| t == positive_class && p == positive_class }
      fn = y_true.zip(y_pred).count { |t, p| t == positive_class && p != positive_class }
      
      tp + fn == 0 ? 0.0 : tp.to_f / (tp + fn)
    end
    
    def self.f1_score(y_true, y_pred, positive_class: 1)
      p = precision(y_true, y_pred, positive_class: positive_class)
      r = recall(y_true, y_pred, positive_class: positive_class)
      
      p + r == 0 ? 0.0 : 2.0 * p * r / (p + r)
    end
    
    def self.classification_report(y_true, y_pred)
      classes = (y_true + y_pred).uniq.sort
      
      report = classes.each_with_object({}) do |cls, hash|
        hash[cls] = {
          precision: precision(y_true, y_pred, positive_class: cls).round(4),
          recall: recall(y_true, y_pred, positive_class: cls).round(4),
          f1_score: f1_score(y_true, y_pred, positive_class: cls).round(4),
          support: y_true.count { |y| y == cls }
        }
      end
      
      report['overall'] = {
        accuracy: accuracy(y_true, y_pred).round(4),
        macro_avg: {
          precision: report.values.sum { |v| v[:precision] } / classes.length,
          recall: report.values.sum { |v| v[:recall] } / classes.length,
          f1_score: report.values.sum { |v| v[:f1_score] } / classes.length
        }
      }
      
      report
    end
    
    # Regression metrics
    def self.mean_squared_error(y_true, y_pred)
      y_true.zip(y_pred).sum { |t, p| (t - p) ** 2 } / y_true.length
    end
    
    def self.root_mean_squared_error(y_true, y_pred)
      Math.sqrt(mean_squared_error(y_true, y_pred))
    end
    
    def self.mean_absolute_error(y_true, y_pred)
      y_true.zip(y_pred).sum { |t, p| (t - p).abs } / y_true.length
    end
    
    def self.r_squared(y_true, y_pred)
      mean = y_true.sum.to_f / y_true.length
      ss_res = y_true.zip(y_pred).sum { |t, p| (t - p) ** 2 }
      ss_tot = y_true.sum { |t| (t - mean) ** 2 }
      
      1 - (ss_res / ss_tot)
    end
  end
end

# ทดสอบ Evaluation Metrics
y_true = [1, 0, 1, 1, 0, 1, 0, 0, 1, 1]
y_pred = [1, 0, 1, 0, 0, 1, 1, 0, 1, 1]

puts "Accuracy: #{ML::Evaluation.accuracy(y_true, y_pred)}"
puts "Precision: #{ML::Evaluation.precision(y_true, y_pred)}"
puts "Recall: #{ML::Evaluation.recall(y_true, y_pred)}"
puts "F1 Score: #{ML::Evaluation.f1_score(y_true, y_pred)}"
puts "\nClassification Report:"
puts ML::Evaluation.classification_report(y_true, y_pred).inspect
```

## 10. Deployment ML Models ใน Rails

```ruby
# config/initializers/ml_models.rb
Rails.application.config.after_initialize do
  unless Rails.env.test?
    # Load ML models on startup
    begin
      Rails.logger.info "Loading ML models..."
      
      # Load spam detector
      SpamDetector.instance
      
      # Schedule periodic model retraining
      if Rails.env.production?
        RetrainModelsJob.set(wait_until: 1.day.from_now).perform_later
      end
      
      Rails.logger.info "ML models loaded successfully"
    rescue => e
      Rails.logger.error "Failed to load ML models: #{e.message}"
    end
  end
end

# app/jobs/retrain_models_job.rb
class RetrainModelsJob < ApplicationJob
  queue_as :ml_training
  
  def perform
    Rails.logger.info "Retraining ML models..."
    
    # Retrain spam detector
    SpamDetector.instance.retrain!
    
    # Retrain recommendation model
    TrainRecommendationModelJob.perform_now
    
    # Schedule next retraining
    RetrainModelsJob.set(wait_until: 1.day.from_now).perform_later
    
    Rails.logger.info "ML models retrained successfully"
  end
end
```

---

## แบบฝึกหัดบทที่ 89

### แบบฝึกหัดที่ 1: Linear Regression
**โจทย์:** สร้าง Multiple Linear Regression ที่รองรับหลาย features และคำนวณ R² score

**เฉลย:**
```ruby
class MultipleLinearRegression
  attr_reader :coefficients, :intercept
  
  def initialize(learning_rate: 0.01, epochs: 1000)
    @learning_rate = learning_rate
    @epochs = epochs
  end
  
  def fit(x_matrix, y_values)
    n_samples = x_matrix.length
    n_features = x_matrix[0].length
    
    @coefficients = Array.new(n_features, 0.0)
    @intercept = 0.0
    
    @epochs.times do
      predictions = x_matrix.map { |x| predict_single(x) }
      errors = predictions.zip(y_values).map { |pred, actual| pred - actual }
      
      # Update intercept
      @intercept -= @learning_rate * (2.0 / n_samples) * errors.sum
      
      # Update coefficients
      n_features.times do |j|
        gradient = (2.0 / n_samples) * x_matrix.zip(errors).sum { |x, e| x[j] * e }
        @coefficients[j] -= @learning_rate * gradient
      end
    end
    
    self
  end
  
  def predict(x_matrix)
    x_matrix.map { |x| predict_single(x) }
  end
  
  def r_squared(x_matrix, y_values)
    predictions = predict(x_matrix)
    mean_y = y_values.sum.to_f / y_values.length
    
    ss_res = predictions.zip(y_values).sum { |pred, actual| (actual - pred) ** 2 }
    ss_tot = y_values.sum { |y| (y - mean_y) ** 2 }
    
    1 - (ss_res / ss_tot)
  end
  
  private
  
  def predict_single(x)
    @intercept + @coefficients.zip(x).sum { |coef, xi| coef * xi }
  end
end

# ทดสอบ
x_train = [[1, 2], [2, 3], [3, 4], [4, 5], [5, 6]]
y_train = [8, 13, 18, 23, 28]

model = MultipleLinearRegression.new(learning_rate: 0.01, epochs: 2000)
model.fit(x_train, y_train)

puts "Coefficients: #{model.coefficients.map { |c| c.round(4) }}"
puts "Intercept: #{model.intercept.round(4)}"
puts "R² Score: #{model.r_squared(x_train, y_train).round(4)}"
puts "Prediction for [6, 7]: #{model.predict([[6, 7]])[0].round(2)}"
```

### แบบฝึกหัดที่ 2: Cross-Validation
**โจทย์:** Implement k-fold cross-validation สำหรับ model evaluation

**เฉลย:**
```ruby
module CrossValidation
  def self.k_fold(features, labels, k: 5, model_class:, **model_params)
    n = features.length
    fold_size = n / k
    
    scores = []
    
    k.times do |fold|
      # Split data
      test_start = fold * fold_size
      test_end = (fold == k - 1) ? n - 1 : test_start + fold_size - 1
      
      test_indices = (test_start..test_end).to_a
      train_indices = (0...n).to_a - test_indices
      
      x_train = train_indices.map { |i| features[i] }
      y_train = train_indices.map { |i| labels[i] }
      x_test = test_indices.map { |i| features[i] }
      y_test = test_indices.map { |i| labels[i] }
      
      # Train and evaluate
      model = model_class.new(**model_params)
      model.fit(x_train, y_train)
      predictions = model.predict(x_test)
      
      accuracy = y_test.zip(predictions).count { |a, p| a == p }.to_f / y_test.length
      scores << accuracy
      
      puts "Fold #{fold + 1}: Accuracy = #{accuracy.round(4)}"
    end
    
    {
      scores: scores,
      mean: scores.sum / scores.length,
      std: Math.sqrt(scores.sum { |s| (s - scores.sum / scores.length) ** 2 } / scores.length)
    }
  end
end

# ทดสอบ
features = Array.new(100) { [rand * 10, rand * 10] }
labels = features.map { |f| f[0] + f[1] > 10 ? 1 : 0 }

results = CrossValidation.k_fold(
  features, labels, k: 5,
  model_class: KNearestNeighbors,
  k: 3
)

puts "\nCross-Validation Results:"
puts "Mean Accuracy: #{results[:mean].round(4)}"
puts "Std Deviation: #{results[:std].round(4)}"
```

### แบบฝึกหัดที่ 3: Feature Importance ด้วย Random Forest Concept
**โจทย์:** Implement feature importance calculation

**เฉลย:**
```ruby
class FeatureImportanceAnalyzer
  def initialize(n_permutations: 10)
    @n_permutations = n_permutations
  end
  
  def calculate_importance(model, features, labels)
    baseline_accuracy = evaluate(model, features, labels)
    importances = {}
    
    features[0].length.times do |feature_idx|
      permuted_accuracies = @n_permutations.times.map do
        # Permute the feature
        permuted_features = features.map.with_index do |f, i|
          f.dup.tap { |pf| pf[feature_idx] = features[rand(features.length)][feature_idx] }
        end
        
        evaluate(model, permuted_features, labels)
      end
      
      avg_permuted = permuted_accuracies.sum / @n_permutations
      importances[feature_idx] = baseline_accuracy - avg_permuted
    end
    
    # Normalize
    total = importances.values.sum
    importances.transform_values { |v| total > 0 ? v / total : 0 }
  end
  
  private
  
  def evaluate(model, features, labels)
    predictions = model.predict(features)
    labels.zip(predictions).count { |a, p| a == p }.to_f / labels.length
  end
end
```

### แบบฝึกหัดที่ 4: Gradient Descent Variants
**โจทย์:** Implement Adam optimizer สำหรับ neural network training

**เฉลย:**
```ruby
class AdamOptimizer
  def initialize(learning_rate: 0.001, beta1: 0.9, beta2: 0.999, epsilon: 1e-8)
    @lr = learning_rate
    @beta1 = beta1
    @beta2 = beta2
    @epsilon = epsilon
    @m = {}  # First moment
    @v = {}  # Second moment
    @t = 0   # Timestep
  end
  
  def update(params, gradients)
    @t += 1
    
    params.each_with_index do |param_group, group_idx|
      grad = gradients[group_idx]
      
      @m[group_idx] ||= Array.new(param_group.length, 0.0)
      @v[group_idx] ||= Array.new(param_group.length, 0.0)
      
      param_group.each_with_index do |param, i|
        # Update biased first moment estimate
        @m[group_idx][i] = @beta1 * @m[group_idx][i] + (1 - @beta1) * grad[i]
        
        # Update biased second raw moment estimate
        @v[group_idx][i] = @beta2 * @v[group_idx][i] + (1 - @beta2) * grad[i] ** 2
        
        # Compute bias-corrected estimates
        m_hat = @m[group_idx][i] / (1 - @beta1 ** @t)
        v_hat = @v[group_idx][i] / (1 - @beta2 ** @t)
        
        # Update parameter
        param_group[i] -= @lr * m_hat / (Math.sqrt(v_hat) + @epsilon)
      end
    end
    
    params
  end
end
```

### แบบฝึกหัดที่ 5: Recommendation Evaluation
**โจทย์:** Implement NDCG (Normalized Discounted Cumulative Gain) สำหรับ recommendation evaluation

**เฉลย:**
```ruby
module RecommendationMetrics
  def self.dcg(relevances, k: nil)
    k ||= relevances.length
    
    relevances.first(k).each_with_index.sum do |rel, i|
      rel / Math.log2(i + 2)
    end
  end
  
  def self.ndcg(predicted_items, relevant_items, k: 10)
    # Get relevance scores for predicted items
    relevances = predicted_items.first(k).map do |item|
      relevant_items.include?(item) ? 1 : 0
    end
    
    # Ideal DCG
    ideal_relevances = [1] * [k, relevant_items.length].min
    
    actual_dcg = dcg(relevances, k: k)
    ideal_dcg = dcg(ideal_relevances, k: k)
    
    return 0.0 if ideal_dcg == 0
    actual_dcg / ideal_dcg
  end
  
  def self.precision_at_k(predicted, relevant, k:)
    relevant_predicted = predicted.first(k) & relevant
    relevant_predicted.length.to_f / k
  end
  
  def self.recall_at_k(predicted, relevant, k:)
    return 0.0 if relevant.empty?
    relevant_predicted = predicted.first(k) & relevant
    relevant_predicted.length.to_f / relevant.length
  end
  
  def self.average_precision(predicted, relevant)
    return 0.0 if relevant.empty?
    
    hits = 0
    sum_precisions = 0.0
    
    predicted.each_with_index do |item, i|
      if relevant.include?(item)
        hits += 1
        sum_precisions += hits.to_f / (i + 1)
      end
    end
    
    sum_precisions / relevant.length
  end
end

# ทดสอบ
predicted = [1, 3, 5, 7, 9, 2, 4, 6, 8, 10]
relevant = [1, 2, 3, 5, 7]

puts "NDCG@5: #{RecommendationMetrics.ndcg(predicted, relevant, k: 5).round(4)}"
puts "NDCG@10: #{RecommendationMetrics.ndcg(predicted, relevant, k: 10).round(4)}"
puts "Precision@5: #{RecommendationMetrics.precision_at_k(predicted, relevant, k: 5).round(4)}"
puts "Recall@5: #{RecommendationMetrics.recall_at_k(predicted, relevant, k: 5).round(4)}"
puts "Average Precision: #{RecommendationMetrics.average_precision(predicted, relevant).round(4)}"
```

### แบบฝึกหัดที่ 6: Online Learning
**โจทย์:** Implement Perceptron algorithm สำหรับ online learning

**เฉลย:**
```ruby
class OnlinePerceptron
  def initialize(n_features, learning_rate: 1.0)
    @weights = Array.new(n_features, 0.0)
    @bias = 0.0
    @learning_rate = learning_rate
    @n_updates = 0
    @n_correct = 0
    @n_total = 0
  end
  
  def update(features, label)
    @n_total += 1
    prediction = classify(features)
    
    if prediction == label
      @n_correct += 1
    else
      @n_updates += 1
      
      # Update weights
      features.each_with_index do |x, i|
        @weights[i] += @learning_rate * label * x
      end
      @bias += @learning_rate * label
    end
    
    prediction
  end
  
  def classify(features)
    score = features.zip(@weights).sum { |x, w| x * w } + @bias
    score >= 0 ? 1 : -1
  end
  
  def accuracy
    @n_total > 0 ? @n_correct.to_f / @n_total : 0.0
  end
  
  def stats
    {
      total_examples: @n_total,
      correct_predictions: @n_correct,
      weight_updates: @n_updates,
      accuracy: accuracy
    }
  end
end
```

### แบบฝึกหัดที่ 7: Ensemble Methods
**โจทย์:** Implement Voting Classifier ที่รวม predictions จากหลาย models

**เฉลย:**
```ruby
class VotingClassifier
  def initialize(estimators:, voting: :hard)
    @estimators = estimators
    @voting = voting
    @trained = false
  end
  
  def fit(features, labels)
    @estimators.each do |name, model|
      puts "Training #{name}..."
      model.fit(features, labels)
    end
    @trained = true
    self
  end
  
  def predict(features)
    raise "Model not trained" unless @trained
    
    if @voting == :hard
      hard_vote(features)
    else
      soft_vote(features)
    end
  end
  
  private
  
  def hard_vote(features)
    predictions_per_model = @estimators.values.map { |model| model.predict(features) }
    
    features.length.times.map do |i|
      votes = predictions_per_model.map { |preds| preds[i] }
      votes.tally.max_by { |_, count| count }[0]
    end
  end
  
  def soft_vote(features)
    # Only works if models support predict_proba
    # Average probabilities
    proba_per_model = @estimators.values.map do |model|
      features.map { |f| model.predict_proba(f) }
    end
    
    features.length.times.map do |i|
      avg_probas = {}
      proba_per_model.each do |model_probas|
        model_probas[i].each do |label, prob|
          avg_probas[label] ||= 0.0
          avg_probas[label] += prob / @estimators.length
        end
      end
      avg_probas.max_by { |_, prob| prob }[0]
    end
  end
end

# ทดสอบ
features = Array.new(100) { [rand * 10, rand * 10] }
labels = features.map { |f| f[0] + f[1] > 10 ? 1 : 0 }

voting_clf = VotingClassifier.new(
  estimators: {
    knn: KNearestNeighbors.new(k: 5),
    dt: DecisionTree.new(max_depth: 5)
  },
  voting: :hard
)

voting_clf.fit(features, labels)
test_features = [[5, 5], [8, 8], [2, 2]]
predictions = voting_clf.predict(test_features)
puts "Voting Classifier Predictions: #{predictions.inspect}"
```

### แบบฝึกหัดที่ 8: Feature Engineering
**โจทย์:** สร้าง Pipeline สำหรับ feature engineering

**เฉลย:**
```ruby
class FeaturePipeline
  def initialize
    @transformers = []
  end
  
  def add_step(name, transformer)
    @transformers << { name: name, transformer: transformer }
    self
  end
  
  def fit(data)
    @transformers.each do |step|
      puts "Fitting #{step[:name]}..."
      step[:transformer].fit(data)
      data = step[:transformer].transform(data)
    end
    self
  end
  
  def transform(data)
    @transformers.inject(data) do |transformed, step|
      step[:transformer].transform(transformed)
    end
  end
  
  def fit_transform(data)
    fit(data).transform(data)
  end
end

class PolynomialFeatures
  def initialize(degree: 2, include_bias: false)
    @degree = degree
    @include_bias = include_bias
  end
  
  def fit(data); self; end
  
  def transform(data)
    data.map do |row|
      features = row.dup
      
      # Add polynomial features
      (2..@degree).each do |d|
        row.each_with_index do |x, i|
          features << x ** d
          
          # Interaction terms
          (i + 1...row.length).each do |j|
            features << x * row[j]
          end
        end
      end
      
      features
    end
  end
end

# ทดสอบ
pipeline = FeaturePipeline.new

data = [[1, 2], [3, 4], [5, 6]]
pipeline.add_step("poly", PolynomialFeatures.new(degree: 2))

transformed = pipeline.fit_transform(data)
puts "Original features: #{data[0].inspect}"
puts "Transformed features: #{transformed[0].inspect}"
```

### แบบฝึกหัดที่ 9: Model Persistence
**โจทย์:** Implement model serialization/deserialization

**เฉลย:**
```ruby
module ModelPersistence
  def save(filename)
    model_data = {
      class_name: self.class.name,
      parameters: serialize_parameters,
      metadata: {
        saved_at: Time.now,
        version: '1.0'
      }
    }
    
    File.write(filename, JSON.generate(model_data))
    puts "Model saved to #{filename}"
  end
  
  def self.load(filename)
    data = JSON.parse(File.read(filename), symbolize_names: true)
    
    model_class = Object.const_get(data[:class_name])
    model = model_class.new
    model.deserialize_parameters(data[:parameters])
    
    puts "Model loaded from #{filename}"
    model
  end
  
  private
  
  def serialize_parameters
    instance_variables.each_with_object({}) do |var, hash|
      hash[var.to_s.delete('@')] = instance_variable_get(var)
    end
  end
end
```

### แบบฝึกหัดที่ 10: Topic Modeling ด้วย LDA
**โจทย์:** Implement Latent Dirichlet Allocation (LDA) อย่างง่าย

**เฉลย:**
```ruby
class SimpleLDA
  def initialize(n_topics:, n_iterations: 100, alpha: 0.1, beta: 0.01)
    @n_topics = n_topics
    @n_iterations = n_iterations
    @alpha = alpha
    @beta = beta
  end
  
  def fit(documents)
    @vocabulary = documents.flatten.uniq
    @vocab_size = @vocabulary.size
    @vocab_index = @vocabulary.each_with_index.to_h
    
    n_docs = documents.length
    
    # Initialize assignments randomly
    @word_topics = documents.map do |doc|
      doc.map { rand(@n_topics) }
    end
    
    # Count matrices
    @doc_topic_counts = Array.new(n_docs) { Array.new(@n_topics, 0) }
    @topic_word_counts = Array.new(@n_topics) { Array.new(@vocab_size, 0) }
    @topic_counts = Array.new(@n_topics, 0)
    
    # Initialize counts
    documents.each_with_index do |doc, d|
      doc.each_with_index do |word, w|
        topic = @word_topics[d][w]
        word_idx = @vocab_index[word]
        
        @doc_topic_counts[d][topic] += 1
        @topic_word_counts[topic][word_idx] += 1
        @topic_counts[topic] += 1
      end
    end
    
    # Gibbs sampling
    @n_iterations.times do |iter|
      documents.each_with_index do |doc, d|
        doc.each_with_index do |word, w|
          current_topic = @word_topics[d][w]
          word_idx = @vocab_index[word]
          
          # Remove word from counts
          @doc_topic_counts[d][current_topic] -= 1
          @topic_word_counts[current_topic][word_idx] -= 1
          @topic_counts[current_topic] -= 1
          
          # Sample new topic
          new_topic = sample_topic(d, word_idx, doc.length)
          @word_topics[d][w] = new_topic
          
          # Add word with new topic
          @doc_topic_counts[d][new_topic] += 1
          @topic_word_counts[new_topic][word_idx] += 1
          @topic_counts[new_topic] += 1
        end
      end
      
      puts "LDA Iteration #{iter + 1}" if (iter + 1) % 10 == 0
    end
    
    self
  end
  
  def top_words_per_topic(n: 10)
    @n_topics.times.map do |topic|
      word_scores = @topic_word_counts[topic].each_with_index.map do |count, word_idx|
        [@vocabulary[word_idx], count]
      end
      
      word_scores.sort_by { |_, count| -count }.first(n)
    end
  end
  
  private
  
  def sample_topic(doc_idx, word_idx, doc_length)
    probabilities = @n_topics.times.map do |topic|
      # P(topic | document) * P(word | topic)
      doc_term = (@doc_topic_counts[doc_idx][topic] + @alpha)
      word_term = (@topic_word_counts[topic][word_idx] + @beta) /
                  (@topic_counts[topic] + @vocab_size * @beta)
      
      doc_term * word_term
    end
    
    # Sample from distribution
    total = probabilities.sum
    rand_val = rand * total
    
    cumulative = 0.0
    probabilities.each_with_index do |prob, topic|
      cumulative += prob
      return topic if rand_val <= cumulative
    end
    
    @n_topics - 1
  end
end
```

### แบบฝึกหัดที่ 11: Anomaly Detection
**โจทย์:** Implement Isolation Forest สำหรับ anomaly detection

**เฉลย:**
```ruby
class IsolationForest
  def initialize(n_trees: 100, sample_size: 256, contamination: 0.1)
    @n_trees = n_trees
    @sample_size = sample_size
    @contamination = contamination
    @trees = []
  end
  
  def fit(data)
    @n_features = data[0].length
    max_depth = Math.log2(@sample_size).ceil
    
    @n_trees.times do
      sample = data.sample(@sample_size)
      @trees << build_tree(sample, 0, max_depth)
    end
    
    # Calculate threshold
    scores = data.map { |point| anomaly_score(point) }
    sorted_scores = scores.sort
    threshold_idx = ((1 - @contamination) * scores.length).round
    @threshold = sorted_scores[threshold_idx]
    
    self
  end
  
  def predict(data)
    data.map { |point| anomaly_score(point) >= @threshold ? -1 : 1 }
  end
  
  def anomaly_score(point)
    avg_depth = @trees.sum { |tree| path_length(point, tree) } / @n_trees.to_f
    c_n = c_factor(@sample_size)
    2 ** (-avg_depth / c_n)
  end
  
  private
  
  def build_tree(data, depth, max_depth)
    return { type: :leaf, size: data.length } if depth >= max_depth || data.length <= 1
    
    feature_idx = rand(@n_features)
    values = data.map { |d| d[feature_idx] }
    min_val = values.min
    max_val = values.max
    
    return { type: :leaf, size: data.length } if min_val == max_val
    
    split_point = min_val + rand * (max_val - min_val)
    
    left_data = data.select { |d| d[feature_idx] < split_point }
    right_data = data.select { |d| d[feature_idx] >= split_point }
    
    {
      type: :node,
      feature_idx: feature_idx,
      split_point: split_point,
      left: build_tree(left_data, depth + 1, max_depth),
      right: build_tree(right_data, depth + 1, max_depth)
    }
  end
  
  def path_length(point, node, depth = 0)
    return depth + c_factor(node[:size]) if node[:type] == :leaf
    
    if point[node[:feature_idx]] < node[:split_point]
      path_length(point, node[:left], depth + 1)
    else
      path_length(point, node[:right], depth + 1)
    end
  end
  
  def c_factor(n)
    return 0 if n <= 1
    2 * (Math.log(n - 1) + 0.5772156649) - (2.0 * (n - 1) / n)
  end
end
```

### แบบฝึกหัดที่ 12: Time Series Prediction
**โจทย์:** Implement ARIMA-like simple time series model

**เฉลย:**
```ruby
class SimpleTimeSeriesModel
  def initialize(window: 5)
    @window = window
    @weights = nil
  end
  
  def fit(series)
    # Create training sequences
    x_sequences = []
    y_values = []
    
    (series.length - @window).times do |i|
      x_sequences << series[i, @window]
      y_values << series[i + @window]
    end
    
    # Fit using linear regression
    n = x_sequences.length
    @weights = Array.new(@window, 0.0)
    @bias = 0.0
    
    100.times do
      predictions = x_sequences.map do |seq|
        seq.zip(@weights).sum { |x, w| x * w } + @bias
      end
      
      errors = predictions.zip(y_values).map { |p, y| p - y }
      
      @window.times do |j|
        gradient = x_sequences.zip(errors).sum { |seq, e| seq[j] * e } / n
        @weights[j] -= 0.001 * gradient
      end
      
      @bias -= 0.001 * errors.sum / n
    end
    
    self
  end
  
  def predict(series, n_steps: 1)
    predictions = []
    current_series = series.last(@window).dup
    
    n_steps.times do
      next_val = current_series.zip(@weights).sum { |x, w| x * w } + @bias
      predictions << next_val
      current_series.shift
      current_series << next_val
    end
    
    predictions
  end
end

# ทดสอบ
series = [1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0, 9.0, 10.0]
model = SimpleTimeSeriesModel.new(window: 3)
model.fit(series)

forecast = model.predict(series, n_steps: 5)
puts "Forecast: #{forecast.map { |v| v.round(2) }.inspect}"
```

### แบบฝึกหัดที่ 13: Word2Vec Concept
**โจทย์:** Implement Simplified Word2Vec (Skip-gram) ด้วย Ruby

**เฉลย:**
```ruby
class SimpleWord2Vec
  def initialize(embedding_dim: 50, window_size: 2, learning_rate: 0.01, epochs: 100)
    @embedding_dim = embedding_dim
    @window_size = window_size
    @lr = learning_rate
    @epochs = epochs
  end
  
  def train(sentences)
    # Build vocabulary
    @vocab = sentences.flatten.uniq.sort
    @word_to_idx = @vocab.each_with_index.to_h
    vocab_size = @vocab.size
    
    # Initialize embeddings
    @w1 = Array.new(vocab_size) { Array.new(@embedding_dim) { (rand * 2 - 1) * 0.1 } }
    @w2 = Array.new(@embedding_dim) { Array.new(vocab_size) { (rand * 2 - 1) * 0.1 } }
    
    @epochs.times do |epoch|
      total_loss = 0.0
      
      sentences.each do |sentence|
        sentence.each_with_index do |center_word, center_idx|
          center_word_idx = @word_to_idx[center_word]
          
          # Get context words
          context_start = [0, center_idx - @window_size].max
          context_end = [sentence.length - 1, center_idx + @window_size].min
          
          (context_start..context_end).each do |context_idx|
            next if context_idx == center_idx
            
            context_word = sentence[context_idx]
            context_word_idx = @word_to_idx[context_word]
            
            # Forward pass
            h = @w1[center_word_idx]
            u = @w2.map { |row| row.zip(h).sum { |w, hi| w * hi } }
            
            # Softmax
            max_u = u.max
            exp_u = u.map { |ui| Math.exp(ui - max_u) }
            sum_exp = exp_u.sum
            y_hat = exp_u.map { |e| e / sum_exp }
            
            # Loss
            total_loss -= Math.log(y_hat[context_word_idx] + 1e-10)
            
            # Backward pass (simplified)
            e = y_hat.dup
            e[context_word_idx] -= 1
            
            # Update w2
            @embedding_dim.times do |k|
              vocab_size.times do |v|
                @w2[k][v] -= @lr * e[v] * h[k]
              end
            end
            
            # Update w1
            grad_h = @embedding_dim.times.map do |k|
              @w2[k].zip(e).sum { |w2_kv, e_v| w2_kv * e_v }
            end
            
            @embedding_dim.times do |k|
              @w1[center_word_idx][k] -= @lr * grad_h[k]
            end
          end
        end
      end
      
      puts "Epoch #{epoch + 1}: Loss = #{(total_loss / sentences.flatten.length).round(4)}" if (epoch + 1) % 10 == 0
    end
    
    self
  end
  
  def word_vector(word)
    idx = @word_to_idx[word]
    return nil unless idx
    @w1[idx]
  end
  
  def most_similar(word, n: 5)
    target_vec = word_vector(word)
    return [] unless target_vec
    
    similarities = @vocab.map do |w|
      next if w == word
      vec = word_vector(w)
      sim = cosine_similarity(target_vec, vec)
      [w, sim]
    end.compact
    
    similarities.sort_by { |_, sim| -sim }.first(n)
  end
  
  private
  
  def cosine_similarity(a, b)
    dot = a.zip(b).sum { |ai, bi| ai * bi }
    mag_a = Math.sqrt(a.sum { |x| x ** 2 })
    mag_b = Math.sqrt(b.sum { |x| x ** 2 })
    
    return 0.0 if mag_a == 0 || mag_b == 0
    dot / (mag_a * mag_b)
  end
end
```

### แบบฝึกหัดที่ 14: Hyperparameter Tuning
**โจทย์:** Implement Grid Search สำหรับ hyperparameter optimization

**เฉลย:**
```ruby
class GridSearchCV
  def initialize(model_class, param_grid:, cv: 5, scoring: :accuracy)
    @model_class = model_class
    @param_grid = param_grid
    @cv = cv
    @scoring = scoring
    @results = []
  end
  
  def fit(features, labels)
    param_combinations = generate_combinations(@param_grid)
    
    param_combinations.each do |params|
      scores = cross_validate(features, labels, params)
      
      @results << {
        params: params,
        mean_score: scores.sum / scores.length,
        std_score: std(scores),
        scores: scores
      }
      
      puts "Params: #{params} => Mean Score: #{(scores.sum / scores.length).round(4)}"
    end
    
    @best_result = @results.max_by { |r| r[:mean_score] }
    puts "\nBest params: #{@best_result[:params]}"
    puts "Best score: #{@best_result[:mean_score].round(4)}"
    
    self
  end
  
  def best_params
    @best_result[:params]
  end
  
  def best_score
    @best_result[:mean_score]
  end
  
  private
  
  def generate_combinations(param_grid)
    keys = param_grid.keys
    values = param_grid.values
    
    combinations = values.reduce([[]], :product).map(&:flatten)
    
    combinations.map do |combo|
      keys.zip(combo).to_h
    end
  end
  
  def cross_validate(features, labels, params)
    n = features.length
    fold_size = n / @cv
    
    @cv.times.map do |fold|
      test_range = (fold * fold_size...(fold + 1) * fold_size)
      test_indices = test_range.to_a
      train_indices = (0...n).to_a - test_indices
      
      x_train = train_indices.map { |i| features[i] }
      y_train = train_indices.map { |i| labels[i] }
      x_test = test_indices.map { |i| features[i] }
      y_test = test_indices.map { |i| labels[i] }
      
      model = @model_class.new(**params)
      model.fit(x_train, y_train)
      predictions = model.predict(x_test)
      
      y_test.zip(predictions).count { |a, p| a == p }.to_f / y_test.length
    end
  end
  
  def std(values)
    mean = values.sum / values.length
    variance = values.sum { |v| (v - mean) ** 2 } / values.length
    Math.sqrt(variance)
  end
end

# ทดสอบ
features = Array.new(200) { [rand * 10, rand * 10] }
labels = features.map { |f| f[0] + f[1] > 10 ? 1 : 0 }

grid_search = GridSearchCV.new(
  KNearestNeighbors,
  param_grid: { k: [1, 3, 5, 7, 9] },
  cv: 5
)

grid_search.fit(features, labels)
puts "Best k: #{grid_search.best_params}"
puts "Best accuracy: #{grid_search.best_score.round(4)}"
```

### แบบฝึกหัดที่ 15: ML Pipeline ใน Rails
**โจทย์:** สร้าง complete ML pipeline ใน Rails สำหรับ sentiment analysis ของ product reviews

**เฉลย:**
```ruby
# app/services/sentiment_analysis_pipeline.rb
class SentimentAnalysisPipeline
  CACHE_KEY = 'sentiment_model'
  
  def initialize
    @vectorizer = TfIdfVectorizer.new(max_features: 5000, min_df: 2)
    @classifier = NaiveBayesClassifier.new
    @trained = false
  end
  
  def self.instance
    @instance ||= new.tap(&:ensure_trained)
  end
  
  def ensure_trained
    if (cached = Rails.cache.read(CACHE_KEY))
      @vectorizer = cached[:vectorizer]
      @classifier = cached[:classifier]
      @trained = true
    else
      train_from_database
    end
  end
  
  def analyze(text)
    raise "Pipeline not trained" unless @trained
    
    vector = @vectorizer.transform([text])[0]
    
    {
      sentiment: @classifier.predict(text),
      confidence: @classifier.predict_proba(text),
      vector_size: vector.length
    }
  end
  
  def batch_analyze(texts)
    texts.map { |text| analyze(text).merge(text: text) }
  end
  
  def retrain!
    train_from_database
  end
  
  private
  
  def train_from_database
    reviews = Review.where.not(sentiment: nil).includes(:product)
    
    return if reviews.count < 100
    
    texts = reviews.map(&:body)
    labels = reviews.map(&:sentiment)  # 'positive', 'negative', 'neutral'
    
    # Fit vectorizer
    vectors = @vectorizer.fit_transform(texts)
    
    # Train classifier
    texts.zip(labels).each do |text, label|
      @classifier.train(text, label)
    end
    
    @trained = true
    
    # Cache the model
    Rails.cache.write(CACHE_KEY, {
      vectorizer: @vectorizer,
      classifier: @classifier
    }, expires_in: 12.hours)
    
    Rails.logger.info "Sentiment model trained on #{reviews.count} reviews"
  end
end

# app/controllers/api/v1/sentiment_controller.rb
module Api
  module V1
    class SentimentController < ApplicationController
      before_action :authenticate_api_user!
      
      def analyze
        text = params[:text]
        
        unless text.present?
          return render json: { error: 'Text is required' }, status: :unprocessable_entity
        end
        
        result = SentimentAnalysisPipeline.instance.analyze(text)
        
        render json: {
          text: text,
          sentiment: result[:sentiment],
          confidence: result[:confidence]
        }
      end
      
      def batch_analyze
        texts = params[:texts]
        
        unless texts.is_a?(Array) && texts.length <= 100
          return render json: { error: 'Provide an array of up to 100 texts' }, 
                       status: :unprocessable_entity
        end
        
        results = SentimentAnalysisPipeline.instance.batch_analyze(texts)
        render json: { results: results }
      end
      
      def model_stats
        stats = {
          total_reviews: Review.count,
          labeled_reviews: Review.where.not(sentiment: nil).count,
          sentiment_distribution: Review.group(:sentiment).count,
          model_trained_at: Rails.cache.read('sentiment_model_trained_at')
        }
        
        render json: stats
      end
    end
  end
end
```

---

## สรุปบทที่ 89

ในบทนี้เราได้เรียนรู้:

1. **ML Fundamentals** - พื้นฐาน machine learning ใน Ruby
2. **Numo::NArray** - สำหรับ numerical computing และ matrix operations
3. **ML Algorithms** - KNN, Decision Tree, K-Means ด้วย pure Ruby
4. **Neural Networks** - Feedforward neural network implementation
5. **Collaborative Filtering** - Recommendation system ทั้ง user-based และ item-based
6. **Matrix Factorization** - SVD-based recommendation
7. **Naive Bayes** - Text classification
8. **TF-IDF** - Feature extraction สำหรับ text
9. **Model Evaluation** - Metrics สำหรับทั้ง classification และ regression
10. **Production Deployment** - Integration กับ Rails application

### ขั้นตอนถัดไป

- ศึกษา Python integration ผ่าน Pycall gem
- ลอง deploy model ด้วย ONNX format
- ศึกษา MLflow สำหรับ experiment tracking
- พัฒนา real-time ML inference ด้วย Sidekiq
