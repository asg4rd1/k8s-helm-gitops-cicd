FROM python:3.11-slim
WORKDIR /app
COPY app.py .
RUN pip install flask --only-binary :all: -r requirements.txt
CMD ["python", "app.py"]
