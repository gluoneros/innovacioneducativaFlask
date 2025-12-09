# innovacioneducativaFlask

A Flask web application for educational innovation with user authentication and database integration.

## Overview

This is a web application built with Flask that provides user registration, login, and dashboard functionality. The application uses PostgreSQL as the database and includes features for managing user accounts securely.

## Features

- User registration with username, email, and password
- Secure user login and session management
- User dashboard with personalized content
- Logout functionality
- Database integration with SQLAlchemy
- User authentication and authorization

## Technologies Used

- Flask - Web framework
- Flask-SQLAlchemy - Database ORM
- Flask-Login - User session management
- PostgreSQL - Database
- Jinja2 - Template engine

## Files Structure

- `main.py` - Main Flask application with routes and models
- `home.html` - Home page template
- `login.html` - Login page template
- `register.html` - Registration page template
- `README.md` - Project documentation

## Setup and Installation

1. Make sure you have Python installed
2. Install dependencies:
   ```bash
   pip install flask flask-sqlalchemy flask-login psycopg2-binary
   ```
3. Set up PostgreSQL database named `innovacioneducativa`
4. Run the application:
   ```bash
   python main.py
   ```

## Database Configuration

The application is configured to connect to a PostgreSQL database with the following settings:
- Host: localhost
- Database name: innovacioneducativa
- Username: usuario
- Password: usuario6

## Routes

- `/` - Home route
- `/register` - User registration
- `/login` - User login
- `/dashboard` - User dashboard (requires authentication)
- `/logout` - User logout
- `/home` - Home page (requires authentication)

## Security Features

- Password hashing for secure storage
- Session management for user authentication
- Login required decorators for protected routes
- CSRF protection through Flask-Login

## Usage

1. Register a new account using the `/register` route
2. Log in using the `/login` route
3. Access the dashboard and home page after authentication