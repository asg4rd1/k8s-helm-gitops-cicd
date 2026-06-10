FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --only-binary :all: -r requirements.txt
COPY app.py .
CMD ["python", "app.py"]