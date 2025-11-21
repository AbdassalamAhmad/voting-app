FROM python:3.14-slim

WORKDIR /app

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

# Install bash, dos2unix, and apache2-utils (for ab)

RUN apt-get update && \
    apt-get install -y apache2-utils && \
    rm -rf /var/lib/apt/lists/*
# Copy scripts
COPY --chown=appuser:appuser . .

# Fix Windows CRLF issue (removes Windows-style line endings (\r) and converts it to Unix-style line endings (\n).)
RUN sed -i 's/\r$//' generate-votes.sh

RUN python3 make-data.py

RUN chmod +x generate-votes.sh

# Switch to non-root user
USER appuser

CMD ["sh", "./generate-votes.sh"]