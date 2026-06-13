
from flask import Flask, render_template, request, jsonify, redirect, make_response, session
from flask_cors import CORS
from flask_sqlalchemy import SQLAlchemy
from werkzeug.security import generate_password_hash, check_password_hash
from datetime import datetime, timedelta
from email.message import EmailMessage
import json
import os
import smtplib
from dotenv import load_dotenv

load_dotenv()

app = Flask(__name__)
app.config['SECRET_KEY'] = os.getenv('SECRET_KEY', 'your-secret-key-change-in-production')
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///autoflow.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
app.config['JSON_SORT_KEYS'] = False
app.config['ADMIN_ACCESS_PASSWORD'] = os.getenv('ADMIN_ACCESS_PASSWORD', 'YourAdminSecret123!')
app.config['ADMIN_USERNAME'] = os.getenv('ADMIN_USERNAME', 'admin123')
app.config['ADMIN_EMAIL'] = os.getenv('ADMIN_EMAIL', 'admin@autoflow.com')
app.config['SMTP_HOST'] = os.getenv('SMTP_HOST')
app.config['SMTP_PORT'] = int(os.getenv('SMTP_PORT', '587'))
app.config['SMTP_USERNAME'] = os.getenv('SMTP_USERNAME')
app.config['SMTP_PASSWORD'] = os.getenv('SMTP_PASSWORD')
app.config['EMAIL_FROM'] = os.getenv('EMAIL_FROM', 'no-reply@autoflow.com')

# Initialize extensions
CORS(app, supports_credentials=True)
db = SQLAlchemy(app)

DATA_FILE = os.path.join(app.root_path, 'static', 'data', 'db.json')

def load_json_db():
    os.makedirs(os.path.dirname(DATA_FILE), exist_ok=True)
    if not os.path.exists(DATA_FILE):
        with open(DATA_FILE, 'w', encoding='utf-8') as f:
            json.dump({'users': [], 'cars': [], 'pendingCars': [], 'bookings': [], 'inquiries': []}, f, indent=2)
    with open(DATA_FILE, 'r', encoding='utf-8') as f:
        return json.load(f)


def save_json_db(data):
    os.makedirs(os.path.dirname(DATA_FILE), exist_ok=True)
    with open(DATA_FILE, 'w', encoding='utf-8') as f:
        json.dump(data, f, indent=2)


def send_verification_email(recipient_email, code):
    host = app.config['SMTP_HOST']
    port = app.config['SMTP_PORT']
    username = app.config['SMTP_USERNAME']
    password = app.config['SMTP_PASSWORD']
    sender = app.config['EMAIL_FROM']

    if not host or not username or not password:
        raise RuntimeError('SMTP server is not configured')

    message = EmailMessage()
    message['Subject'] = 'AutoFlow Verification Code'
    message['From'] = sender
    message['To'] = recipient_email
    message.set_content(
        f'Your AutoFlow verification code is: {code}\n\n'
        'This code expires in 10 minutes. Do not share it with anyone.'
    )

    smtp_class = smtplib.SMTP_SSL if port == 465 else smtplib.SMTP
    with smtp_class(host, port) as smtp:
        if port != 465:
            smtp.starttls()
        smtp.login(username, password)
        smtp.send_message(message)


def get_insurance_by_name(name):
    return Insurance.query.filter_by(name=name).first()


def get_insurance_levels():
    return [insurance.to_dict() for insurance in Insurance.query.order_by(Insurance.name).all()]


def is_admin_session():
    return session.get('role') == 'admin' or request.cookies.get('role') == 'admin'


def ensure_admin_user():
    admin_username = app.config['ADMIN_USERNAME']
    admin_password = app.config['ADMIN_ACCESS_PASSWORD']
    admin_email = app.config['ADMIN_EMAIL']
    admin_user = User.query.filter_by(username=admin_username).first()
    if not admin_user:
        admin_user = User.query.filter_by(email=admin_email).first()

    if not admin_user:
        admin_user = User(
            username=admin_username,
            email=admin_email,
            full_name='System Administrator',
            phone='+000000000000',
            location='Headquarters',
            is_dealer=False,
            is_verified=True
        )
        admin_user.set_password(admin_password)
        db.session.add(admin_user)
        db.session.commit()
    else:
        changed = False
        if admin_user.username != admin_username:
            admin_user.username = admin_username
            changed = True
        if admin_user.email != admin_email:
            admin_user.email = admin_email
            changed = True
        if not admin_user.check_password(admin_password):
            admin_user.set_password(admin_password)
            changed = True
        if not admin_user.is_verified:
            admin_user.is_verified = True
            changed = True
        if changed:
            db.session.commit()


class Booking(db.Model):
    __tablename__ = 'bookings'
    id = db.Column(db.Integer, primary_key=True)
    car_id = db.Column(db.Integer, db.ForeignKey('cars.id'), nullable=False)
    customer_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
    start_date = db.Column(db.DateTime)
    end_date = db.Column(db.DateTime)
    insurance = db.Column(db.String(100))
    status = db.Column(db.String(50), default='pending review')
    created_at = db.Column(db.DateTime, default=datetime.utcnow)


class Review(db.Model):
    __tablename__ = 'reviews'
    id = db.Column(db.Integer, primary_key=True)
    booking_id = db.Column(db.Integer, db.ForeignKey('bookings.id'))
    car_id = db.Column(db.Integer, db.ForeignKey('cars.id'))
    rating = db.Column(db.Integer)
    comment = db.Column(db.Text)
    reviewer_id = db.Column(db.Integer, db.ForeignKey('users.id'))
    reviewed_user_id = db.Column(db.Integer, db.ForeignKey('users.id'))
    created_at = db.Column(db.DateTime, default=datetime.utcnow)


class User(db.Model):
    __tablename__ = 'users'
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(200), nullable=False)
    full_name = db.Column(db.String(120), nullable=False)
    phone = db.Column(db.String(20), nullable=False)
    profile_pic = db.Column(db.String(200))
    is_dealer = db.Column(db.Boolean, default=False)
    location = db.Column(db.String(200))
    rating = db.Column(db.Float, default=5.0)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    is_verified = db.Column(db.Boolean, default=False)

    cars = db.relationship('Car', backref='owner', lazy=True, cascade='all, delete-orphan')
    bookings_made = db.relationship('Booking', backref='customer', lazy=True, foreign_keys='Booking.customer_id')
    reviews_given = db.relationship('Review', backref='reviewer', lazy=True, foreign_keys='Review.reviewer_id')
    reviews_received = db.relationship('Review', backref='reviewed_user', lazy=True, foreign_keys='Review.reviewed_user_id')

    def set_password(self, password):
        self.password_hash = generate_password_hash(password)

    def check_password(self, password):
        return check_password_hash(self.password_hash, password)

    def to_dict(self):
        return {
            'id': self.id,
            'username': self.username,
            'email': self.email,
            'full_name': self.full_name,
            'phone': self.phone,
            'is_dealer': self.is_dealer,
            'location': self.location,
            'rating': self.rating,
            'created_at': self.created_at.isoformat() if self.created_at else None
        }


class Car(db.Model):
    __tablename__ = 'cars'
    id = db.Column(db.Integer, primary_key=True)
    owner_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
    model_name = db.Column(db.String(120), nullable=False)
    brand = db.Column(db.String(100), nullable=False)
    year = db.Column(db.Integer, nullable=False)
    car_type = db.Column(db.String(50), nullable=False)
    fuel_type = db.Column(db.String(50), nullable=False)
    daily_rate = db.Column(db.Float, nullable=False)
    location = db.Column(db.String(200), nullable=False)
    latitude = db.Column(db.Float)
    longitude = db.Column(db.Float)
    description = db.Column(db.Text)
    image_url = db.Column(db.String(500))
    mileage = db.Column(db.Integer)
    seats = db.Column(db.Integer, default=5)
    transmission = db.Column(db.String(50))
    color = db.Column(db.String(50))
    is_available = db.Column(db.Boolean, default=True)
    rating = db.Column(db.Float, default=5.0)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    updated_at = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    bookings = db.relationship('Booking', backref='car', lazy=True, cascade='all, delete-orphan')
    reviews = db.relationship('Review', backref='car', lazy=True, cascade='all, delete-orphan')

    MULTIPLIERS = {
        'Sedan': 1.0,
        'SUV': 1.5,
        '4x4': 1.8,
        'Luxury': 2.5,
        'Electric': 1.3
    }

    def get_adjusted_daily_rate(self):
        base_rate = 5000
        multiplier = self.MULTIPLIERS.get(self.car_type, 1.0)
        return base_rate * multiplier

    def to_dict(self):
        return {
            'id': self.id,
            'owner_id': self.owner_id,
            'model_name': self.model_name,
            'brand': self.brand,
            'year': self.year,
            'car_type': self.car_type,
            'fuel_type': self.fuel_type,
            'daily_rate': self.daily_rate,
            'location': self.location,
            'latitude': self.latitude,
            'longitude': self.longitude,
            'description': self.description,
            'image_url': self.image_url,
            'mileage': self.mileage,
            'seats': self.seats,
            'transmission': self.transmission,
            'color': self.color,
            'is_available': self.is_available,
            'rating': self.rating,
            'created_at': self.created_at.isoformat() if self.created_at else None,
            'updated_at': self.updated_at.isoformat() if self.updated_at else None
        }


class Insurance(db.Model):
    __tablename__ = 'insurance'
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), unique=True, nullable=False)
    daily_cost = db.Column(db.Float, nullable=False)
    coverage = db.Column(db.Text)
    description = db.Column(db.Text)

    def to_dict(self):
        return {
            'id': self.id,
            'name': self.name,
            'daily_cost': self.daily_cost,
            'coverage': self.coverage,
            'description': self.description
        }


with app.app_context():
    db.create_all()
    try:
        ensure_admin_user()
    except Exception as err:
        print(f'Admin user initialization failed: {err}')


@app.route('/api/auth/request-verification', methods=['POST'])
def request_user_verification():
    # No verification required - skipped for simplified auth
    return jsonify({'message': 'Ready to register', 'demo_code': '111111'}), 200


@app.route('/api/auth/register', methods=['POST'])
def register():
    try:
        payload = request.get_json() or {}
        mandatory = ['username', 'email', 'password', 'full_name', 'phone']
        missing = [field for field in mandatory if not payload.get(field)]
        if missing:
            return jsonify({'error': f'Missing fields: {", ".join(missing)}'}), 400

        username = payload['username'].strip()
        email = payload['email'].strip().lower()
        phone = payload['phone'].strip()
        password = payload['password']

        if username.lower() == 'admin':
            return jsonify({'error': 'Admin account creation not allowed'}), 403

        if User.query.filter_by(username=username).first():
            return jsonify({'error': 'Username already taken'}), 409
        if User.query.filter_by(email=email).first():
            return jsonify({'error': 'Email already exists'}), 409

        user = User(
            username=username,
            email=email,
            full_name=payload['full_name'].strip(),
            phone=phone,
            location=payload.get('location', ''),
            is_dealer=bool(payload.get('is_dealer', False)),
            is_verified=True
        )
        user.set_password(password)

        db.session.add(user)
        db.session.commit()
        session.pop('pending_user_verification', None)

        return jsonify({
            'message': 'Account created successfully',
            'user': user.to_dict(),
            'role': 'user'
        }), 201

    except Exception as err:
        print(f"Registration error: {err}")
        db.session.rollback()
        return jsonify({'error': 'Registration failed'}), 500


@app.route('/api/auth/login', methods=['POST'])
def login():
    try:
        payload = request.get_json() or {}
        username = (payload.get('username') or '').strip()
        password = payload.get('password') or ''

        if not username or not password:
            return jsonify({'error': 'Username and password required'}), 400

        admin_username = app.config['ADMIN_USERNAME']
        admin_password = app.config['ADMIN_ACCESS_PASSWORD']
        
        # Check if admin credentials
        if username.lower() == admin_username.lower() and password == admin_password:
            role = 'admin'
            user_data = {'username': admin_username, 'email': app.config['ADMIN_EMAIL'], 'role': 'admin'}
        else:
            # Check SQL users - no verification required
            sql_user = User.query.filter_by(username=username).first()
            if not sql_user or not sql_user.check_password(password):
                return jsonify({'error': 'Invalid credentials'}), 401
            role = 'dealer' if sql_user.is_dealer else 'user'
            user_data = sql_user.to_dict()

        resp = make_response(jsonify({'message': 'Login successful', 'role': role, 'user': user_data}))
        resp.set_cookie('role', role, httponly=True, path='/')
        resp.set_cookie('username', username, httponly=True, path='/')
        session['role'] = role
        session['username'] = username
        return resp, 200

    except Exception as err:
        print(f"Login error: {err}")
        return jsonify({'error': 'Login failed'}), 500


@app.route('/api/current-user', methods=['GET'])
def get_current_user():
    return jsonify({
        'role': session.get('role') or 'guest',
        'username': session.get('username') or ''
    }), 200


@app.route('/api/stats/dashboard', methods=['GET'])
def get_dashboard_stats():
    return jsonify({
        'available_cars': Car.query.filter_by(is_available=True).count(),
        'total_cars': Car.query.count(),
        'total_bookings': Booking.query.count(),
        'total_users': User.query.count()
    }), 200


@app.route('/api/users', methods=['GET'])
def get_all_users():
    users = User.query.all()
    return jsonify({'users': [u.to_dict() for u in users]}), 200


@app.route('/api/cars', methods=['GET'])
def get_all_cars():
    cars = Car.query.all()
    return jsonify({'cars': [c.to_dict() for c in cars]}), 200


@app.route('/api/bookings', methods=['GET'])
def get_all_bookings():
    bookings = Booking.query.all()
    return jsonify({'bookings': [
        {
            'id': b.id,
            'car_id': b.car_id,
            'customer_id': b.customer_id,
            'start_date': b.start_date.isoformat() if b.start_date else None,
            'end_date': b.end_date.isoformat() if b.end_date else None,
            'insurance_type': b.insurance,
            'insurance_cost': 0,
            'status': b.status
        }
        for b in bookings
    ]}), 200


@app.route('/api/db/book', methods=['POST'])
def api_book_car():
    payload = request.get_json() or {}
    required_fields = ['car_id', 'customer_name', 'email', 'phone', 'start_date', 'end_date', 'insurance']
    if not all(field in payload and payload.get(field) for field in required_fields):
        return jsonify({'error': 'Please complete all booking fields.'}), 400

    start_date = payload.get('start_date')
    end_date = payload.get('end_date')
    try:
        if datetime.fromisoformat(start_date) > datetime.fromisoformat(end_date):
            return jsonify({'error': 'End date must be after start date.'}), 400
    except Exception:
        return jsonify({'error': 'Invalid date format.'}), 400

    raw_data = load_json_db()
    car = next((item for item in raw_data.get('cars', []) if item['id'] == payload['car_id']), None)
    if not car:
        return jsonify({'error': 'Selected car not found.'}), 404

    insurance_type = payload['insurance']
    insurance = get_insurance_by_name(insurance_type)
    if not insurance:
        return jsonify({'error': 'Selected insurance level is not available'}), 400

    try:
        start_dt = datetime.fromisoformat(start_date)
        end_dt = datetime.fromisoformat(end_date)
    except Exception:
        return jsonify({'error': 'Invalid date format.'}), 400

    days = max((end_dt - start_dt).days, 1)
    insurance_cost = insurance.daily_cost * days
    total_cost = car['daily_rate'] * days + insurance_cost

    booking_id = max([b['id'] for b in raw_data.get('bookings', [])] or [0]) + 1
    booking_record = {
        'id': booking_id,
        'car_id': payload['car_id'],
        'car_title': f"{car.get('brand')} {car.get('model_name')}",
        'customer_name': payload['customer_name'],
        'email': payload['email'],
        'phone': payload['phone'],
        'start_date': payload['start_date'],
        'end_date': payload['end_date'],
        'insurance': insurance_type,
        'insurance_cost': insurance_cost,
        'days': days,
        'total_cost': total_cost,
        'status': 'pending review',
        'created_at': datetime.utcnow().isoformat()
    }

    raw_data.setdefault('bookings', []).append(booking_record)
    save_json_db(raw_data)
    return jsonify({'message': 'Booking successfully submitted for dealer evaluation', 'booking': booking_record}), 201
    if not insurance:
        return jsonify({'error': 'Selected insurance level is not available'}), 400

    try:
        start_dt = datetime.fromisoformat(start_date)
        end_dt = datetime.fromisoformat(end_date)
    except Exception:
        return jsonify({'error': 'Invalid date format.'}), 400

    days = max((end_dt - start_dt).days, 1)
    insurance_cost = insurance.daily_cost * days
    total_cost = car['daily_rate'] * days + insurance_cost

    booking_record = {
        'id': booking_id,
        'car_id': payload['car_id'],
        'car_title': f"{car.get('brand')} {car.get('model_name')}",
        'customer_name': payload['customer_name'],
        'email': payload['email'],
        'phone': payload['phone'],
        'start_date': payload['start_date'],
        'end_date': payload['end_date'],
        'insurance': insurance_type,
        'insurance_cost': insurance_cost,
        'days': days,
        'total_cost': total_cost,
        'status': 'pending review',
        'created_at': datetime.utcnow().isoformat()
    }

    raw_data.setdefault('bookings', []).append(booking_record)
    save_json_db(raw_data)
    return jsonify({'message': 'Booking successfully submitted for dealer evaluation', 'booking': booking_record}), 201


@app.route('/api/insurance', methods=['GET'])
def list_insurance_levels():
    return jsonify(get_insurance_levels()), 200


@app.route('/api/insurance', methods=['POST'])
def create_insurance_level():
    if not is_admin_session():
        return jsonify({'error': 'Admin access required to add insurance levels'}), 401

    payload = request.get_json() or {}
    required = ['name', 'daily_cost', 'coverage', 'description']
    if not all(payload.get(field) for field in required):
        return jsonify({'error': 'Missing required insurance fields'}), 400

    name = payload['name'].strip()
    if get_insurance_by_name(name):
        return jsonify({'error': 'Insurance level already exists'}), 409

    try:
        daily_cost = float(payload['daily_cost'])
    except ValueError:
        return jsonify({'error': 'Invalid daily_cost value'}), 400

    insurance = Insurance(
        name=name,
        daily_cost=daily_cost,
        coverage=payload['coverage'].strip(),
        description=payload['description'].strip()
    )
    db.session.add(insurance)
    db.session.commit()
    return jsonify({'message': 'Insurance level added', 'insurance': insurance.to_dict()}), 201


@app.route('/api/db/register', methods=['POST'])
def api_db_register():
    payload = request.get_json() or {}
    mandatory = ['username', 'password', 'name', 'email', 'phone']
    
    if not all(payload.get(field) for field in mandatory):
        return jsonify({'error': 'Required form parameters missing'}), 400

    if payload.get('username', '').strip().lower() == 'admin' or payload.get('role') == 'admin':
        return jsonify({'error': 'Cannot create admin account through this endpoint'}), 403

    raw_data = load_json_db()
    users_list = raw_data.setdefault('users', [])
    
    # Duplicate username/email validations
    if any(u['username'].lower() == payload['username'].strip().lower() for u in users_list):
        return jsonify({'error': 'Selected username is not available'}), 409
    if any(u['email'].lower() == payload['email'].strip().lower() for u in users_list):
        return jsonify({'error': 'An account is already linked to this email'}), 409

    role_val = payload.get('role', 'user')
    if role_val not in ['user', 'dealer']:
        role_val = 'user'

    next_id = max([u['id'] for u in users_list] or [0]) + 1
    new_user = {
        'id': next_id,
        'username': payload['username'].strip(),
        'password': payload['password'],
        'role': role_val,
        'name': payload['name'].strip(),
        'email': payload['email'].strip(),
        'phone': payload['phone'].strip()
    }
    
    users_list.append(new_user)
    save_json_db(raw_data)
    return jsonify({'message': 'User profile successfully registered to file'}), 201


@app.route('/api/db/login', methods=['POST'])
def api_db_login():
    payload = request.get_json() or {}
    username = payload.get('username', '').strip()
    password = payload.get('password', '').strip()
    
    if not username or not password:
        return jsonify({'error': 'Username and password credentials required'}), 400

    role = None
    user_data = None
    raw_data = load_json_db()
    matched_user = next((u for u in raw_data.get('users', []) if u['username'] == username and u['password'] == password), None)

    if matched_user:
        role = matched_user.get('role', 'user')
        user_data = matched_user
    else:
        # First, support the configured admin credentials even if the admin user is not stored in the JSON file.
        admin_username = app.config['ADMIN_USERNAME']
        if username.lower() == admin_username.lower() and password == app.config['ADMIN_ACCESS_PASSWORD']:
            role = 'admin'
            user_data = {
                'username': admin_username,
                'email': app.config['ADMIN_EMAIL'],
                'role': 'admin'
            }
        else:
            # Fallback to SQL user auth for other users and verified admin accounts.
            sql_user = User.query.filter_by(username=username).first()
            if sql_user and sql_user.check_password(password):
                if not sql_user.is_verified:
                    return jsonify({'error': 'Email address has not been verified'}), 403
                role = 'admin' if username.lower() == admin_username.lower() else ('dealer' if sql_user.is_dealer else 'user')
                user_data = sql_user.to_dict()

    if not role:
        return jsonify({'error': 'Login credentials incorrect'}), 401

    resp = make_response(jsonify({'message': 'Authenticated', 'role': role, 'user': user_data}))
    resp.set_cookie('role', role, httponly=True, path='/')
    resp.set_cookie('username', username, httponly=True, path='/')
    session['role'] = role
    session['username'] = username
    return resp, 200


@app.route('/logout', methods=['GET'])
def logout():
    session.clear()
    resp = make_response(redirect('/'))
    resp.set_cookie('role', '', expires=0, path='/')
    resp.set_cookie('username', '', expires=0, path='/')
    return resp


# ==========================================
# DATABASE INITIALIZER ENDPOINT
# ==========================================

@app.route('/api/init-db', methods=['POST'])
@app.route('/init-db', methods=['POST'])
def init_db():
    try:
        db.drop_all()
        db.create_all()
        
        # 1. Add baseline users
        admin_account = User(
            username=app.config['ADMIN_USERNAME'],
            email=app.config['ADMIN_EMAIL'],
            full_name='System Admin',
            phone='+92 300 0000000',
            location='Main Headquarters',
            is_dealer=True,
            is_verified=True
        )
        admin_account.set_password(app.config['ADMIN_ACCESS_PASSWORD'])
        
        individual_owner = User(
            username='ahmed_owner',
            email='ahmed@example.com',
            full_name='Ahmed Hassan',
            phone='+92 300 1234567',
            location='Lahore',
            is_dealer=False
        )
        individual_owner.set_password('password123')
        
        dealer_owner = User(
            username='dealer_pk',
            email='dealer@example.com',
            full_name='Professional Dealers',
            phone='+92 321 9876543',
            location='Islamabad',
            is_dealer=True
        )
        dealer_owner.set_password('password123')
        
        db.session.add_all([admin_account, individual_owner, dealer_owner])
        db.session.commit()
        
        # 2. Add baseline vehicles
        car1 = Car(
            owner_id=individual_owner.id,
            model_name='Honda Civic',
            brand='Honda',
            year=2023,
            car_type='Sedan',
            fuel_type='Petrol',
            daily_rate=4500,
            location='5km away from UET Lahore',
            description='Well-maintained Honda Civic in excellent condition',
            seats=5,
            transmission='Automatic',
            color='Silver'
        )
        
        car2 = Car(
            owner_id=dealer_owner.id,
            model_name='Toyota Fortuner',
            brand='Toyota',
            year=2022,
            car_type='SUV',
            fuel_type='Diesel',
            daily_rate=7500,
            location='3km away from UET Lahore',
            description='Spacious SUV perfect for group travels',
            seats=7,
            transmission='Automatic',
            color='Black'
        )
        
        car3 = Car(
            owner_id=dealer_owner.id,
            model_name='Tesla Model 3',
            brand='Tesla',
            year=2024,
            car_type='Electric',
            fuel_type='Electric',
            daily_rate=8000,
            location='2km away from UET Lahore',
            description='Eco-friendly electric vehicle with premium features',
            seats=5,
            transmission='Automatic',
            color='White'
        )
        
        db.session.add_all([car1, car2, car3])
        db.session.commit()
        
        # 3. Add active insurance tiers
        insurance_tiers = [
            Insurance(
                name='No Insurance',
                daily_cost=0.0,
                coverage='No coverage',
                description='No insurance included'
            ),
            Insurance(
                name='Basic',
                daily_cost=500.0,
                coverage='Basic damage coverage up to PKR 50,000',
                description='Basic insurance coverage'
            ),
            Insurance(
                name='Premium',
                daily_cost=1000.0,
                coverage='Full damage coverage up to PKR 200,000',
                description='Premium insurance with comprehensive coverage'
            ),
            Insurance(
                name='Full Protection',
                daily_cost=1500.0,
                coverage='Complete coverage including accidents and theft',
                description='Full protection plan with zero deductible'
            )
        ]
        
        db.session.add_all(insurance_tiers)
        db.session.commit()
        
        return jsonify({
            'message': 'Database schemas and data initialized successfully',
            'users': 3,
            'cars': 3,
            'insurances': 4
        }), 201
    except Exception as err:
        db.session.rollback()
        print(f"Error resetting database schemas: {err}")
        return jsonify({'error': f'Failed to reset database: {str(err)}'}), 500


# ==========================================
# PAGE ROUTINGS (JINJA2 RENDERING)
# ==========================================

@app.route('/', methods=['GET'])
def home():
    # Renders the home catalog search index.html page
    return render_template('index.html')


@app.route('/login', methods=['GET'])
def login_page():
    # Renders registration and login login.html page
    return render_template('login.html')


@app.route('/admin', methods=['GET'])
def admin_page():
    if not is_admin_session():
        return redirect('/login')
    return render_template('admin.html')


# ==========================================
# MAIN EXECUTION ENTRYPOINT
# ==========================================

if __name__ == '__main__':
    with app.app_context():
        # Setup tables if not already existing
        db.create_all()
    app.run(debug=True, host='0.0.0.0', port=5000)
