FROM python:3.11-slim

# Установка системных зависимостей
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    curl \
    libexpat1 \
    && rm -rf /var/lib/apt/lists/*

# Создание рабочей директории
WORKDIR /app

# Копирование requirements.txt и установка зависимостей
COPY requirements.txt .
RUN pip install --upgrade pip \
    && pip install --no-cache-dir -r requirements.txt

# Копирование исходного кода
COPY . .

# Экспонирование порта Streamlit
EXPOSE 8501

# Команда запуска
CMD ["streamlit", "run", "loader.py", "--server.port=8501", "--server.address=0.0.0.0"]
