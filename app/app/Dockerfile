FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
# 🔑 Add the port your app listens on (e.g., Flask defaults to 5000)
EXPOSE 5000
CMD ["python", "app.py"]

