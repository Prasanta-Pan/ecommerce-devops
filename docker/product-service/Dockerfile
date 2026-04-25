# ── Stage 1: Build ────────────────────────────────────────────────────────────
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy solution and project files first for better layer caching
COPY ProductService.sln .
COPY ProductService.API/ProductService.API.csproj ProductService.API/
COPY ProductService.Service/ProductService.Service.csproj ProductService.Service/
COPY ProductService.Data/ProductService.Data.csproj ProductService.Data/
COPY ProductService.Tests/ProductService.Tests.csproj ProductService.Tests/

# Restore NuGet packages
RUN dotnet restore ProductService.sln

# Copy all remaining source files
COPY . .

# Build
RUN dotnet build ProductService.sln -c Release --no-restore

# Run tests
RUN dotnet test ProductService.Tests/ProductService.Tests.csproj \
    -c Release \
    --no-build \
    --verbosity normal

# Publish
RUN dotnet publish ProductService.API/ProductService.API.csproj \
    -c Release \
    --no-build \
    -o /app/publish

# ── Stage 2: Runtime ──────────────────────────────────────────────────────────
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app

# Create a non-root user
RUN addgroup --system --gid 1000 appgroup \
    && adduser --system --uid 1000 --ingroup appgroup --no-create-home appuser

# Copy published output
COPY --from=build /app/publish .

# Set ownership
RUN chown -R appuser:appgroup /app

USER appuser

# Expose port
EXPOSE 8080

# Set ASP.NET Core to listen on port 8080
ENV ASPNETCORE_URLS=http://+:8080
ENV DOTNET_RUNNING_IN_CONTAINER=true

ENTRYPOINT ["dotnet", "ProductService.API.dll"]
