FROM mcr.microsoft.com/dotnet/sdk:6.0-alpine AS builder

WORKDIR /src
COPY src/Numbers.Api/Numbers.Api.csproj .
RUN dotnet restore

COPY src/Numbers.Api/ .
RUN dotnet publish -c Release -o /out Numbers.Api.csproj

# app image
FROM mcr.microsoft.com/dotnet/aspnet:6.0-alpine

ENV FailAfterCallCount=0

ENTRYPOINT ["dotnet", "/app/Numbers.Api.dll"]

WORKDIR /app
COPY --from=builder /out/ .