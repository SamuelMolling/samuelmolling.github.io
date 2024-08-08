---
title: "Todo List in Golang"
excerpt: "A Todo List application implemented in Golang using MongoDB for data storage. <br/><img src='/images/portfolio/todo-list.png' style='width:300px;'>"
collection: portfolio
tags:
  - mongodb
  - golang
---
Todo List in Golang is a simple yet powerful Todo List application implemented in Go (Golang) using MongoDB for data storage. This project showcases essential concepts of web development with Go, including routing, JSON handling, and integration with a NoSQL database.

## Features:
* CRUD Operations: Supports Create, Read, Update, and Delete operations for managing tasks.
* MongoDB Integration: Utilizes MongoDB Atlas for storing tasks, leveraging secure cloud connection.
* Gin Framework: Uses the Gin framework to create fast and efficient HTTP APIs.
* CORS Middleware: Implements middleware to allow cross-origin requests (CORS).

## How to Use:
1. **Setup:** Ensure you have Go installed on your system. Clone the repository from GitHub.
```bash
git clone https://github.com/SamuelMolling/todo-list-golang.git
cd todo-list-golang
```
2. **MongoDB Configuration:** Set up your cluster and configure your connection string on line 32.
```bash
mongoURI := "mongodb+srv://<USERNAME>:<PASSWORD>@<CLUTER_ENDPOINT>/?retryWrites=true&w=majority&appName=demo1"
```
3. Run the Application: Execute the following command to start the server.
```bash
go run main.go
```
4. Access the Application: Open your browser and navigate to `http://localhost:8080` to interact with the Todo List application.

[More information here](https://github.com/SamuelMolling/todo-list-golang)