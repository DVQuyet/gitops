FROM python:3.12-slim

RUN pip install flask prometheus-flask-exporter

COPY app.py /app/app.py
WORKDIR /app

ENV FLASK_APP=app.py

EXPOSE 9090

CMD ["flask", "run", "--host=0.0.0.0", "--port=9090"]
