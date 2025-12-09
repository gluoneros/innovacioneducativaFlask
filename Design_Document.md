# Innovacioneducativa Backend Technical Design Document

## 1. Executive Summary

Innovacioneducativa is a RESTful backend system designed for a School Management System (SMS) with a Flutter frontend. The system follows a modular architecture using Flask with the Application Factory pattern, PostgreSQL database, and JWT-based authentication. This document outlines the technical architecture, data models, and implementation guidelines for the core modules.

## 2. Architecture Overview

### 2.1 Tech Stack
- **Language**: Python 3.10+
- **Framework**: Flask with Application Factory Pattern
- **ORM**: SQLAlchemy
- **Database**: PostgreSQL
- **Serialization**: Marshmallow
- **Authentication**: Flask-JWT-Extended
- **Migrations**: Flask-Migrate

### 2.2 Project Structure
```
innovacioneducaiva/
├── app/
│   ├── __init__.py                 # Application Factory
│   ├── extensions.py               # Flask extensions initialization
│   ├── models/                     # SQLAlchemy models
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── course.py
│   │   ├── grade.py
│   │   └── relationship.py
│   ├── schemas/                    # Marshmallow schemas
│   │   ├── __init__.py
│   │   ├── user_schema.py
│   │   ├── course_schema.py
│   │   └── grade_schema.py
│   ├── modules/                    # Feature modules (Blueprints)
│   │   ├── auth/                   # Authentication module
│   │   │   ├── __init__.py
│   │   │   ├── routes.py
│   │   │   └── utils.py
│   │   ├── users/                  # User management module
│   │   │   ├── __init__.py
│   │   │   ├── routes.py
│   │   │   └── utils.py
│   │   ├── courses/                # Course management module
│   │   │   ├── __init__.py
│   │   │   ├── routes.py
│   │   │   └── utils.py
│   │   ├── grades/                 # Grade management module
│   │   │   ├── __init__.py
│   │   │   ├── routes.py
│   │   │   └── utils.py
│   │   └── common/                 # Common utilities
│   │       ├── __init__.py
│   │       ├── decorators.py       # Role-based decorators
│   │       └── errors.py           # Global error handlers
├── migrations/                     # Database migrations
├── tests/                          # Test files
├── config.py                       # Configuration settings
├── requirements.txt                # Dependencies
├── docker-compose.yml              # Docker configuration
└── run.py                          # Application entry point
```

### 2.3 Scalability Architecture
The modular blueprint structure allows for easy expansion of features:
- **AI Module**: Analytics and predictive modeling
- **Video Module**: Video streaming and conferencing
- **Payments Module**: Financial transactions
- **Communication Module**: Notifications and messaging

Each new module follows the same blueprint pattern, maintaining consistency and separation of concerns.

## 3. Data Model Design (ERD Logical)

### 3.1 User Model
```python
class User(db.Model):
    __tablename__ = 'users'
    
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(255), nullable=False)
    first_name = db.Column(db.String(50), nullable=False)
    last_name = db.Column(db.String(50), nullable=False)
    role = db.Column(db.Enum('student', 'teacher', 'admin', 'guardian', name='user_roles'), nullable=False)
    date_of_birth = db.Column(db.Date)
    phone = db.Column(db.String(20))
    address = db.Column(db.Text)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    updated_at = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    is_active = db.Column(db.Boolean, default=True)
    
    # Relationships
    grades = db.relationship('Grade', backref='student', lazy=True, foreign_keys='Grade.student_id')
    courses_taught = db.relationship('Course', backref='teacher', lazy=True, foreign_keys='Course.teacher_id')
    student_courses = db.relationship('Course', secondary='student_courses', back_populates='students')
    guardian_students = db.relationship('User', 
                                       secondary='guardian_student',
                                       primaryjoin='User.id == guardian_student.c.guardian_id',
                                       secondaryjoin='User.id == guardian_student.c.student_id',
                                       backref='guardians')
```

### 3.2 Course Model
```python
class Course(db.Model):
    __tablename__ = 'courses'
    
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    code = db.Column(db.String(20), unique=True, nullable=False)
    description = db.Column(db.Text)
    credits = db.Column(db.Integer, default=3)
    teacher_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    updated_at = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    # Many-to-Many relationship with Students
    students = db.relationship('User', secondary='student_courses', back_populates='student_courses')
    grades = db.relationship('Grade', backref='course', lazy=True)
```

### 3.3 Grade Model
```python
class Grade(db.Model):
    __tablename__ = 'grades'
    
    id = db.Column(db.Integer, primary_key=True)
    student_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
    course_id = db.Column(db.Integer, db.ForeignKey('courses.id'), nullable=False)
    teacher_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)  # Who assigned the grade
    assignment_name = db.Column(db.String(100), nullable=False)
    score = db.Column(db.Float, nullable=False)  # 0.0 to 100.0 or 0.0 to 5.0
    max_score = db.Column(db.Float, default=100.0)
    date_assigned = db.Column(db.DateTime, default=datetime.utcnow)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    updated_at = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    # Ensure unique assignment per student per course
    __table_args__ = (db.UniqueConstraint('student_id', 'course_id', 'assignment_name', name='unique_student_assignment'),)
```

### 3.4 Relationship Tables
```python
# Many-to-Many: Students and Courses
student_courses = db.Table('student_courses',
    db.Column('student_id', db.Integer, db.ForeignKey('users.id'), primary_key=True),
    db.Column('course_id', db.Integer, db.ForeignKey('courses.id'), primary_key=True)
)

# Many-to-Many: Guardians and Students
guardian_student = db.Table('guardian_student',
    db.Column('guardian_id', db.Integer, db.ForeignKey('users.id'), primary_key=True),
    db.Column('student_id', db.Integer, db.ForeignKey('users.id'), primary_key=True)
)
```

## 4. Core Implementation

### 4.1 Configuration (config.py)
```python
import os
from datetime import timedelta

class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'dev-secret-key-change-in-production'
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or 'postgresql://user:password@localhost/educore'
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    JWT_SECRET_KEY = os.environ.get('JWT_SECRET_KEY') or 'jwt-secret-string-change-in-production'
    JWT_ACCESS_TOKEN_EXPIRES = timedelta(hours=1)
    JWT_REFRESH_TOKEN_EXPIRES = timedelta(days=30)

class DevelopmentConfig(Config):
    DEBUG = True
    SQLALCHEMY_DATABASE_URI = os.environ.get('DEV_DATABASE_URL') or 'postgresql://user:password@localhost/educore_dev'

class ProductionConfig(Config):
    DEBUG = False
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL')

class TestingConfig(Config):
    TESTING = True
    SQLALCHEMY_DATABASE_URI = os.environ.get('TEST_DATABASE_URL') or 'postgresql://user:password@localhost/educore_test'

config = {
    'development': DevelopmentConfig,
    'production': ProductionConfig,
    'testing': TestingConfig,
    'default': DevelopmentConfig
}
```

### 4.2 Application Factory (app/__init__.py)
```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_jwt_extended import JWTManager
from flask_marshmallow import Marshmallow
from app.extensions import db, migrate, jwt, ma
from config import config

def create_app(config_name='default'):
    app = Flask(__name__)
    app.config.from_object(config[config_name])
    
    # Initialize extensions
    db.init_app(app)
    migrate.init_app(app, db)
    jwt.init_app(app)
    ma.init_app(app)
    
    # Register blueprints
    from app.modules.auth import auth_bp
    app.register_blueprint(auth_bp, url_prefix='/api/auth')
    
    from app.modules.users import users_bp
    app.register_blueprint(users_bp, url_prefix='/api/users')
    
    from app.modules.courses import courses_bp
    app.register_blueprint(courses_bp, url_prefix='/api/courses')
    
    from app.modules.grades import grades_bp
    app.register_blueprint(grades_bp, url_prefix='/api/grades')
    
    # Register error handlers
    from app.modules.common.errors import register_error_handlers
    register_error_handlers(app)
    
    return app
```

### 4.3 User Model with Password Hashing (app/models/user.py)
```python
from app.extensions import db
from werkzeug.security import generate_password_hash, check_password_hash
from datetime import datetime
from enum import Enum

class UserRole(Enum):
    STUDENT = 'student'
    TEACHER = 'teacher'
    ADMIN = 'admin'
    GUARDIAN = 'guardian'

class User(db.Model):
    __tablename__ = 'users'
    
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(255), nullable=False)
    first_name = db.Column(db.String(50), nullable=False)
    last_name = db.Column(db.String(50), nullable=False)
    role = db.Column(db.Enum(UserRole, name='user_roles'), nullable=False)
    date_of_birth = db.Column(db.Date)
    phone = db.Column(db.String(20))
    address = db.Column(db.Text)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    updated_at = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    is_active = db.Column(db.Boolean, default=True)
    
    # Relationships
    grades = db.relationship('Grade', backref='student', lazy=True, foreign_keys='Grade.student_id')
    courses_taught = db.relationship('Course', backref='teacher', lazy=True, foreign_keys='Course.teacher_id')
    student_courses = db.relationship('Course', secondary='student_courses', back_populates='students')
    guardian_students = db.relationship('User', 
                                       secondary='guardian_student',
                                       primaryjoin='User.id == guardian_student.c.guardian_id',
                                       secondaryjoin='User.id == guardian_student.c.student_id',
                                       backref='guardians')
    
    def set_password(self, password):
        self.password_hash = generate_password_hash(password)
    
    def check_password(self, password):
        return check_password_hash(self.password_hash, password)
    
    def to_dict(self):
        return {
            'id': self.id,
            'username': self.username,
            'email': self.email,
            'first_name': self.first_name,
            'last_name': self.last_name,
            'role': self.role.value,
            'date_of_birth': self.date_of_birth.isoformat() if self.date_of_birth else None,
            'phone': self.phone,
            'address': self.address,
            'created_at': self.created_at.isoformat(),
            'updated_at': self.updated_at.isoformat(),
            'is_active': self.is_active
        }
```

### 4.4 Authentication Module (app/modules/auth/routes.py)
```python
from flask import Blueprint, request, jsonify
from flask_jwt_extended import create_access_token, create_refresh_token, jwt_required, get_jwt_identity
from app.models.user import User, db
from app.schemas.user_schema import user_schema

auth_bp = Blueprint('auth', __name__)

@auth_bp.route('/register', methods=['POST'])
def register():
    data = request.get_json()
    
    # Validate required fields
    required_fields = ['username', 'email', 'password', 'first_name', 'last_name', 'role']
    for field in required_fields:
        if field not in data:
            return {'message': f'Missing {field} field'}, 400
    
    # Check if user already exists
    if User.query.filter_by(username=data['username']).first():
        return {'message': 'Username already exists'}, 409
    
    if User.query.filter_by(email=data['email']).first():
        return {'message': 'Email already exists'}, 409
    
    # Create new user
    user = User(
        username=data['username'],
        email=data['email'],
        first_name=data['first_name'],
        last_name=data['last_name'],
        role=data['role']
    )
    user.set_password(data['password'])
    
    db.session.add(user)
    db.session.commit()
    
    return {'message': 'User created successfully'}, 201

@auth_bp.route('/login', methods=['POST'])
def login():
    data = request.get_json()
    
    if not data or not data.get('username') or not data.get('password'):
        return {'message': 'Username and password required'}, 400
    
    user = User.query.filter_by(username=data['username']).first()
    
    if not user or not user.check_password(data['password']):
        return {'message': 'Invalid credentials'}, 401
    
    if not user.is_active:
        return {'message': 'Account deactivated'}, 401
    
    access_token = create_access_token(identity=user.id)
    refresh_token = create_refresh_token(identity=user.id)
    
    return {
        'access_token': access_token,
        'refresh_token': refresh_token,
        'user': user.to_dict()
    }, 200

@auth_bp.route('/refresh', methods=['POST'])
@jwt_required(refresh=True)
def refresh():
    current_user_id = get_jwt_identity()
    user = User.query.get(current_user_id)
    
    if not user or not user.is_active:
        return {'message': 'User not found or inactive'}, 401
    
    new_token = create_access_token(identity=current_user_id)
    return {'access_token': new_token}, 200
```

### 4.5 Grade Management with Role-Based Access (app/modules/grades/routes.py)
```python
from flask import Blueprint, request, jsonify
from flask_jwt_extended import jwt_required, get_jwt_identity
from app.models.user import User, UserRole
from app.models.grade import Grade, db
from app.modules.common.decorators import admin_required, teacher_required, student_required
from app.schemas.grade_schema import grade_schema, grades_schema

grades_bp = Blueprint('grades', __name__)

@grades_bp.route('/', methods=['GET'])
@jwt_required()
def get_grades():
    current_user_id = get_jwt_identity()
    current_user = User.query.get(current_user_id)
    
    if current_user.role == UserRole.STUDENT:
        # Students can only see their own grades
        grades = Grade.query.filter_by(student_id=current_user_id).all()
        return grades_schema.dump(grades), 200
    elif current_user.role == UserRole.TEACHER:
        # Teachers can see grades for courses they teach
        grades = Grade.query.filter_by(teacher_id=current_user_id).all()
        return grades_schema.dump(grades), 200
    elif current_user.role == UserRole.GUARDIAN:
        # Guardians can see grades for their students
        student_ids = [student.id for student in current_user.guardian_students]
        grades = Grade.query.filter(Grade.student_id.in_(student_ids)).all()
        return grades_schema.dump(grades), 200
    elif current_user.role == UserRole.ADMIN:
        # Admins can see all grades
        grades = Grade.query.all()
        return grades_schema.dump(grades), 200
    else:
        return {'message': 'Insufficient permissions'}, 403

@grades_bp.route('/<int:grade_id>', methods=['GET'])
@jwt_required()
def get_grade(grade_id):
    grade = Grade.query.get_or_404(grade_id)
    current_user_id = get_jwt_identity()
    current_user = User.query.get(current_user_id)
    
    # Check permissions based on user role
    if current_user.role == UserRole.STUDENT and grade.student_id != current_user_id:
        return {'message': 'Access denied'}, 403
    elif current_user.role == UserRole.TEACHER and grade.teacher_id != current_user_id:
        return {'message': 'Access denied'}, 403
    elif current_user.role == UserRole.GUARDIAN:
        if grade.student_id not in [s.id for s in current_user.guardian_students]:
            return {'message': 'Access denied'}, 403
    elif current_user.role != UserRole.ADMIN:
        return {'message': 'Insufficient permissions'}, 403
    
    return grade_schema.dump(grade), 200

@grades_bp.route('/', methods=['POST'])
@teacher_required
def create_grade():
    data = request.get_json()
    
    required_fields = ['student_id', 'course_id', 'assignment_name', 'score']
    for field in required_fields:
        if field not in data:
            return {'message': f'Missing {field} field'}, 400
    
    # Verify that the teacher teaches this course
    current_user_id = get_jwt_identity()
    if data['course_id'] not in [c.id for c in User.query.get(current_user_id).courses_taught]:
        return {'message': 'You can only assign grades to courses you teach'}, 403
    
    grade = Grade(
        student_id=data['student_id'],
        course_id=data['course_id'],
        teacher_id=current_user_id,
        assignment_name=data['assignment_name'],
        score=data['score'],
        max_score=data.get('max_score', 100.0)
    )
    
    db.session.add(grade)
    db.session.commit()
    
    return grade_schema.dump(grade), 201

@grades_bp.route('/<int:grade_id>', methods=['PUT'])
@teacher_required
def update_grade(grade_id):
    grade = Grade.query.get_or_404(grade_id)
    
    # Verify that the teacher teaches this course
    current_user_id = get_jwt_identity()
    if grade.teacher_id != current_user_id:
        return {'message': 'You can only modify grades you assigned'}, 403
    
    data = request.get_json()
    
    # Update allowed fields
    if 'assignment_name' in data:
        grade.assignment_name = data['assignment_name']
    if 'score' in data:
        grade.score = data['score']
    if 'max_score' in data:
        grade.max_score = data['max_score']
    
    db.session.commit()
    
    return grade_schema.dump(grade), 200

@grades_bp.route('/<int:grade_id>', methods=['DELETE'])
@admin_required
def delete_grade(grade_id):
    grade = Grade.query.get_or_404(grade_id)
    db.session.delete(grade)
    db.session.commit()
    
    return {'message': 'Grade deleted successfully'}, 200
```

### 4.6 Role-Based Decorators (app/modules/common/decorators.py)
```python
from functools import wraps
from flask import jsonify
from flask_jwt_extended import verify_jwt_in_request, get_jwt_identity
from app.models.user import User, UserRole

def admin_required(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        verify_jwt_in_request()
        current_user_id = get_jwt_identity()
        current_user = User.query.get(current_user_id)
        
        if not current_user or current_user.role != UserRole.ADMIN:
            return {'message': 'Admin access required'}, 403
        
        return fn(*args, **kwargs)
    return wrapper

def teacher_required(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        verify_jwt_in_request()
        current_user_id = get_jwt_identity()
        current_user = User.query.get(current_user_id)
        
        if not current_user or current_user.role not in [UserRole.TEACHER, UserRole.ADMIN]:
            return {'message': 'Teacher access required'}, 403
        
        return fn(*args, **kwargs)
    return wrapper

def student_required(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        verify_jwt_in_request()
        current_user_id = get_jwt_identity()
        current_user = User.query.get(current_user_id)
        
        if not current_user or current_user.role not in [UserRole.STUDENT, UserRole.ADMIN]:
            return {'message': 'Student access required'}, 403
        
        return fn(*args, **kwargs)
    return wrapper

def guardian_required(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        verify_jwt_in_request()
        current_user_id = get_jwt_identity()
        current_user = User.query.get(current_user_id)
        
        if not current_user or current_user.role not in [UserRole.GUARDIAN, UserRole.ADMIN]:
            return {'message': 'Guardian access required'}, 403
        
        return fn(*args, **kwargs)
    return wrapper
```

### 4.7 Global Error Handlers (app/modules/common/errors.py)
```python
from flask import jsonify
from werkzeug.exceptions import HTTPException
from app import db

def register_error_handlers(app):
    @app.errorhandler(400)
    def bad_request(error):
        return jsonify({'message': 'Bad request', 'error': str(error)}), 400

    @app.errorhandler(401)
    def unauthorized(error):
        return jsonify({'message': 'Unauthorized access'}), 401

    @app.errorhandler(403)
    def forbidden(error):
        return jsonify({'message': 'Forbidden access'}), 403

    @app.errorhandler(404)
    def not_found(error):
        return jsonify({'message': 'Resource not found'}), 404

    @app.errorhandler(405)
    def method_not_allowed(error):
        return jsonify({'message': 'Method not allowed'}), 405

    @app.errorhandler(500)
    def internal_error(error):
        db.session.rollback()
        return jsonify({'message': 'Internal server error'}), 500

    @app.errorhandler(Exception)
    def handle_exception(e):
        if isinstance(e, HTTPException):
            return e
        app.logger.error(f'Unhandled exception: {str(e)}')
        return jsonify({'message': 'An unexpected error occurred'}), 500
```

## 5. Local Deployment Instructions

### 5.1 Prerequisites
- Python 3.10+
- PostgreSQL
- Docker (optional, for database container)

### 5.2 Setup Steps

1. **Create Virtual Environment**
```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

2. **Install Dependencies**
```bash
pip install -r requirements.txt
```

3. **Database Setup**
```bash
# Create database (if not using Docker)
createdb educore_dev

# Or use Docker for PostgreSQL
docker-compose up -d
```

4. **Environment Configuration**
```bash
export SECRET_KEY="your-secret-key-here"
export DATABASE_URL="postgresql://user:password@localhost/educore_dev"
export JWT_SECRET_KEY="your-jwt-secret-here"
```

5. **Run Migrations**
```bash
flask db init
flask db migrate -m "Initial migration"
flask db upgrade
```

6. **Start the Application**
```bash
python run.py
```

### 5.3 Docker Compose for PostgreSQL (docker-compose.yml)
```yaml
version: '3.8'
services:
  postgres:
    image: postgres:14
    environment:
      POSTGRES_DB: educore_dev
      POSTGRES_USER: educore_user
      POSTGRES_PASSWORD: educore_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### 5.4 Requirements File (requirements.txt)
```
Flask==2.3.3
Flask-SQLAlchemy==3.0.5
Flask-Migrate==4.0.5
Flask-JWT-Extended==4.5.3
Flask-Marshmallow==0.15.0
marshmallow-sqlalchemy==0.29.0
psycopg2-binary==2.9.7
python-dotenv==1.0.0
Werkzeug==2.3.7
```

## 6. API Response Format

All API endpoints return standardized JSON responses:

### Success Response
```json
{
  "data": { ... },
  "message": "Operation successful",
  "status_code": 200
}
```

### Error Response
```json
{
  "message": "Error description",
  "status_code": 400,
  "error": "Optional error details"
}
```

## 7. Security Considerations

- JWT tokens with proper expiration times
- Passwords hashed using Werkzeug's secure hashing
- Role-based access control for all endpoints
- Input validation and sanitization
- SQL injection prevention through ORM
- Rate limiting (to be implemented)
- HTTPS in production

## 8. Future Enhancements

- Integration with AI for predictive analytics
- Real-time notifications using WebSockets
- File upload for assignments and documents
- Payment processing module
- Video conferencing integration
- Advanced reporting and analytics