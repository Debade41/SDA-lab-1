FROM python:3.13-alpine
WORKDIR /app
COPY --chown=65534:65534 server.py .
USER 65534:65534
CMD ["python", "-u", "server.py"]
