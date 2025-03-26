# ADDBAR1
/sdcard/AgriVisionAI/
   ├── app.py          (# app.py
from flask import Flask, request, jsonify
import tensorflow as tf
from PIL import Image
import numpy as np
import requests
import io
import sqlite3

app = Flask(__name__)

# Load plant identification model (pretrained MobileNetV2)
PLANT_MODEL = tf.keras.models.load_model('plant_identification_model.h5')
PLANT_CLASSES = ['tomato', 'lettuce', 'basil', 'strawberry', 'pepper']  # Example classes

# Database setup for farm records
def init_db():
    conn = sqlite3.connect('farm_data.db')
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS plants
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                  plant_type TEXT,
                  health_status TEXT,
                  detection_time TIMESTAMP,
                  image_path TEXT)''')
    conn.commit()
    conn.close()

# Plant identification endpoint
@app.route('/identify', methods=['POST'])
def identify_plant():
    if 'image' not in request.files:
        return jsonify({'error': 'No image provided'}), 400
    
    img_file = request.files['image']
    img = Image.open(io.BytesIO(img_file.read()))
    img = img.resize((224, 224))  # Model expects 224x224
    
    # Preprocess image
    img_array = tf.keras.preprocessing.image.img_to_array(img)
    img_array = tf.expand_dims(img_array, 0)
    img_array = img_array / 255.0
    
    # Make prediction
    predictions = PLANT_MODEL.predict(img_array)
    predicted_class = PLANT_CLASSES[np.argmax(predictions[0])]
    confidence = float(np.max(predictions[0]))
    
    # Check for pests/diseases
    pest_detection = detect_pests(img)
    
    # Store in database
    conn = sqlite3.connect('farm_data.db')
    c = conn.cursor()
    c.execute("INSERT INTO plants (plant_type, health_status, detection_time) VALUES (?, ?, datetime('now'))",
              (predicted_class, pest_detection['status']))
    conn.commit()
    conn.close()
    
    return jsonify({
        'plant_type': predicted_class,
        'confidence': confidence,
        'pest_detection': pest_detection
    })

# Pest detection function (simplified)
def detect_pests(img):
    # In a real app, this would use a separate trained model
    # Here we'll simulate with some basic checks
    
    # Convert to numpy array
    img_array = np.array(img)
    
    # Check for common color patterns indicating disease
    mean_color = np.mean(img_array, axis=(0,1))
    
    alerts = []
    status = "healthy"
    
    # Yellowing leaves detection (simplified)
    if mean_color[1] > 150 and mean_color[0] > 120 and mean_color[2] < 100:
        alerts.append("Possible nutrient deficiency (yellowing leaves)")
        status = "needs_attention"
    
    # Dark spots detection
    if np.any(img_array[:,:,0] < 50) and np.any(img_array[:,:,1] < 50):
        alerts.append("Possible fungal infection (dark spots detected)")
        status = "needs_attention"
    
    return {
        'status': status,
        'alerts': alerts,
        'recommendations': [
            "Check nutrient levels",
            "Monitor plant daily",
            "Isolate if infection suspected"
        ]
    }

if __name__ == '__main__':
    init_db()
    app.run(host='0.0.0.0', port=5000))
   ├── mobile-app/
   │   └── App.js      (// App.js
import React, { useState } from 'react';
import { View, Text, Button, Image, StyleSheet, Alert } from 'react-native';
import { launchCamera } from 'react-native-image-picker';

const AgriVisionApp = () => {
  const [image, setImage] = useState(null);
  const [result, setResult] = useState(null);
  const [loading, setLoading] = useState(false);

  const takePhoto = async () => {
    const options = {
      mediaType: 'photo',
      quality: 0.8,
    };

    launchCamera(options, (response) => {
      if (response.didCancel) {
        console.log('User cancelled image picker');
      } else if (response.error) {
        console.log('ImagePicker Error: ', response.error);
      } else {
        setImage(response.assets[0].uri);
        analyzePlant(response.assets[0]);
      }
    });
  };

  const analyzePlant = async (imageData) => {
    setLoading(true);
    
    const formData = new FormData();
    formData.append('image', {
      uri: imageData.uri,
      type: imageData.type || 'image/jpeg',
      name: imageData.fileName || 'plant_image.jpg',
    });

    try {
      const response = await fetch('http://YOUR_SERVER_IP:5000/identify', {
        method: 'POST',
        body: formData,
        headers: {
          'Content-Type': 'multipart/form-data',
        },
      });
      
      const data = await response.json();
      setResult(data);
      
      // Show alerts if any issues detected
      if (data.pest_detection.status !== 'healthy') {
        Alert.alert(
          'Plant Health Alert',
          data.pest_detection.alerts.join('\n'),
          [
            { text: 'OK', onPress: () => console.log('Alert closed') }
          ]
        );
      }
    } catch (error) {
      console.error('Error:', error);
      Alert.alert('Error', 'Failed to analyze plant');
    } finally {
      setLoading(false);
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>AgriVision AI</Text>
      
      <Button 
        title="Scan Plant" 
        onPress={takePhoto} 
        disabled={loading}
      />
      
      {image && (
        <Image source={{ uri: image }} style={styles.image} />
      )}
      
      {loading && <Text>Analyzing plant...</Text>}
      
      {result && (
        <View style={styles.resultContainer}>
          <Text>Plant Type: {result.plant_type}</Text>
          <Text>Confidence: {(result.confidence * 100).toFixed(1)}%</Text>
          <Text>Status: {result.pest_detection.status}</Text>
          
          {result.pest_detection.alerts.length > 0 && (
            <View style={styles.alertContainer}>
              <Text style={styles.alertTitle}>Alerts:</Text>
              {result.pest_detection.alerts.map((alert, index) => (
                <Text key={index}>- {alert}</Text>
              ))}
            </View>
          )}
        </View>
      )}
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 20,
    alignItems: 'center',
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    marginBottom: 20,
  },
  image: {
    width: 300,
    height: 300,
    marginVertical: 20,
    resizeMode: 'contain',
  },
  resultContainer: {
    marginTop: 20,
    padding: 15,
    backgroundColor: '#f0f0f0',
    borderRadius: 10,
    width: '100%',
  },
  alertContainer: {
    marginTop: 10,
    padding: 10,
    backgroundColor: '#ffeeee',
    borderRadius: 5,
  },
  alertTitle: {
    fontWeight: 'bold',
    color: 'red',
  },
});

export default AgriVisionApp;)
   └── assets/         (# train_model.py
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import layers
from tensorflow.keras.preprocessing.image import ImageDataGenerator

# Parameters
IMG_SIZE = (224, 224)
BATCH_SIZE = 32
EPOCHS = 10

# Data generators
train_datagen = ImageDataGenerator(
    rescale=1./255,
    rotation_range=20,
    width_shift_range=0.2,
    height_shift_range=0.2,
    shear_range=0.2,
    zoom_range=0.2,
    horizontal_flip=True,
    validation_split=0.2
)

train_generator = train_datagen.flow_from_directory(
    'dataset/plants',
    target_size=IMG_SIZE,
    batch_size=BATCH_SIZE,
    class_mode='categorical',
    subset='training'
)

val_generator = train_datagen.flow_from_directory(
    'dataset/plants',
    target_size=IMG_SIZE,
    batch_size=BATCH_SIZE,
    class_mode='categorical',
    subset='validation'
)

# Model architecture (transfer learning)
base_model = tf.keras.applications.MobileNetV2(
    input_shape=(224, 224, 3),
    include_top=False,
    weights='imagenet'
)

base_model.trainable = False

model = keras.Sequential([
    base_model,
    layers.GlobalAveragePooling2D(),
    layers.Dense(128, activation='relu'),
    layers.Dropout(0.2),
    layers.Dense(len(train_generator.class_indices), activation='softmax')
])

model.compile(
    optimizer='adam',
    loss='categorical_crossentropy',
    metrics=['accuracy']
)

# Train model
history = model.fit(
    train_generator,
    epochs=EPOCHS,
    validation_data=val_generator
)

# Save model
model.save('plant_identification_model.h5'))
   
