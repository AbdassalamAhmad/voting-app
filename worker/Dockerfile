FROM mcr.microsoft.com/dotnet/sdk:7.0 as builder

WORKDIR /src
COPY Worker.csproj .
RUN dotnet restore

COPY . .
RUN dotnet publish -c Release -o /app/publish

# Runtime stage
FROM mcr.microsoft.com/dotnet/runtime:7.0

WORKDIR /app

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

# Copy published application
COPY --from=builder --chown=appuser:appuser /app/publish .

# Switch to non-root user
USER appuser

CMD ["dotnet", "Worker.dll"]