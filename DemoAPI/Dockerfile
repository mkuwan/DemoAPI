#See https://aka.ms/containerfastmode to understand how Visual Studio uses this Dockerfile to build your images for faster debugging.

FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
WORKDIR /app
EXPOSE 5001


ENV ASPNETCORE_URLS=http://+:5001

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["DemoAPI/DemoAPI.csproj", "DemoAPI/"]
RUN dotnet restore "DemoAPI/DemoAPI.csproj"
COPY . .
WORKDIR "/src/DemoAPI"
RUN dotnet build "DemoAPI.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "DemoAPI.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "DemoAPI.dll"]