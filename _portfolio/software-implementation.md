---
title: "Location system with Flask and Bank system with Tkinter"
excerpt: "Two simplified applications implemented for a software implementation class project. <br/><img src='/images/portfolio/python-learning.png' style='width:300px;'>"
collection: portfolio
tags:
  - flask
  - tkinter
  - python
  - sqlite
---
# GA and GB Projects in Flask and Tkinter

**GA and GB Projects** are two simplified applications developed for a software implementation class project. These projects showcase fundamental concepts of web and desktop application development using Flask and Tkinter, respectively, with persistent data storage using SQLite.

## GA Project: Bank System

This software is a banking system simplified using Python 3.10 with Tkinter and persistency using SQLite3.

### Features:
* **Account Management:** Allows users to create and manage bank accounts.
* **Transaction Handling:** Enables users to perform deposits, withdrawals, and view transaction history.
* **Database Integration:** Uses SQLite3 for storing account and transaction data.
* **Desktop Interface:** Built with Tkinter for a simple and user-friendly desktop application.

### How to Use GA Project:
1. **Setup:** Ensure you have Python and Tkinter installed on your system. Clone the repository from GitHub.
    ```bash
    git clone https://github.com/SamuelMolling/software-implementation-projects.git
    cd software-implementation-projects/gaProject
    ```

2. **Install Dependencies:** Install the required dependencies using pip.
    ```bash
    pip install -r requirements.txt
    ```
3. **Run the Application:** Execute the following command to start the Tkinter application.
    ```bash
    python3 main.py
    ```

## GB Project: 

This software is a location system simplified using Python 3.10 with Flask and persistency using SQLite3.

### Features:
* **User Management:** Allows users to register and log in.
* **Location Tracking:** Enables users to add and view locations.
* **Database Integration:** Uses SQLite3 for storing user information and location data.
* **Web Interface:** Built with Flask and Jinja2 for dynamic HTML rendering.

### How to Use GB Project:
1. **Setup:** Ensure you have Python and Flask installed on your system. Clone the repository from GitHub.
   ```bash
   git clone https://github.com/SamuelMolling/software-implementation-projects.git
   cd software-implementation-projects/gbProject
   ```
2. **Install Dependencies:** Install the required dependencies using pip.
    ```bash
    pip install -r requirements.txt
    ```

3. **Create Database:** Create your data if not exists.
    ```bash
    python3 models.py
    ```

4. **Run the Application:** Execute the following command to start the Flask server.
    ```bash
    python3 app.py
    ```

5. **Access the Application:** Open your browser and navigate to http://localhost:5000 to interact with the location system.

[More information here](https://github.com/SamuelMolling/software-implementation-projects)