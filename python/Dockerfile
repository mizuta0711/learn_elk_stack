FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# コンテナ起動時は何もしない（手動実行用）
CMD ["tail", "-f", "/dev/null"]
