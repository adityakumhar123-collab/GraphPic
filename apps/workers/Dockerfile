FROM python:3.11-slim
WORKDIR /app

# Install system dependencies required by OpenCV and InsightFace
RUN apt-get update && apt-get install -y \
    libgl1-mesa-glx \
    libglib2.0-0 \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies (use image_worker's requirements — both are identical)
COPY image_worker/requirements.txt ./requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

# Copy all worker source
COPY . .

# Run both workers as background processes
CMD ["sh", "-c", "python image_worker/worker.py & python query_worker/worker.py & wait"]
