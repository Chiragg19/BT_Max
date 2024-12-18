from flask import Flask, request, jsonify
import uuid
from typing import Dict, List

# Initialize Flask app
app = Flask(__name__)

# In-memory storage for books and members
books_db: Dict[int, 'Book'] = {}
members_db: Dict[int, 'Member'] = {}

# Token-based authentication setup
valid_tokens = {}

# Constants for pagination
BOOKS_PER_PAGE = 5
MEMBERS_PER_PAGE = 5


# Model for Book and Member
class Book:
    def __init__(self, id: int, title: str, author: str, year: int):
        self.id = id
        self.title = title
        self.author = author
        self.year = year

    def to_dict(self):
        return {'id': self.id, 'title': self.title, 'author': self.author, 'year': self.year}


class Member:
    def __init__(self, id: int, name: str, email: str):
        self.id = id
        self.name = name
        self.email = email

    def to_dict(self):
        return {'id': self.id, 'name': self.name, 'email': self.email}


# Helper function for pagination
def paginate(data, page, per_page):
    start = (page - 1) * per_page
    end = start + per_page
    return list(data.values())[start:end]


# Token Generation & Validation
def generate_token():
    token = str(uuid.uuid4())
    valid_tokens[token] = True
    return token


def token_required(f):
    def wrapper(*args, **kwargs):
        token = request.headers.get('Authorization')
        if token and valid_tokens.get(token):
            return f(*args, **kwargs)
        else:
            return jsonify({"message": "Unauthorized"}), 401
    return wrapper


# Authentication Route (for demo purposes)
@app.route('/auth/login', methods=['POST'])
def login():
    token = generate_token()
    return jsonify({"token": token})


# Book Routes

@app.route('/books', methods=['POST'])
@token_required
def add_book():
    data = request.json
    new_id = len(books_db) + 1
    book = Book(id=new_id, title=data['title'], author=data['author'], year=data['year'])
    books_db[new_id] = book
    return jsonify(book.to_dict()), 201


@app.route('/books', methods=['GET'])
@token_required
def get_books():
    page = int(request.args.get('page', 1))
    books = paginate(books_db, page, BOOKS_PER_PAGE)
    return jsonify([book.to_dict() for book in books])


@app.route('/books/<int:id>', methods=['GET'])
@token_required
def get_book(id):
    book = books_db.get(id)
    if not book:
        return jsonify({"message": "Book not found"}), 404
    return jsonify(book.to_dict())


@app.route('/books/<int:id>', methods=['PUT'])
@token_required
def update_book(id):
    data = request.json
    book = books_db.get(id)
    if not book:
        return jsonify({"message": "Book not found"}), 404
    book.title = data['title']
    book.author = data['author']
    book.year = data['year']
    return jsonify(book.to_dict())


@app.route('/books/<int:id>', methods=['DELETE'])
@token_required
def delete_book(id):
    book = books_db.pop(id, None)
    if not book:
        return jsonify({"message": "Book not found"}), 404
    return jsonify({"message": "Book deleted"}), 200


@app.route('/books/search', methods=['GET'])
@token_required
def search_books():
    query = request.args.get('query', '').lower()
    filtered_books = [
        book for book in books_db.values() if query in book.title.lower() or query in book.author.lower()
    ]
    return jsonify([book.to_dict() for book in filtered_books])


# Member Routes

@app.route('/members', methods=['POST'])
@token_required
def add_member():
    data = request.json
    new_id = len(members_db) + 1
    member = Member(id=new_id, name=data['name'], email=data['email'])
    members_db[new_id] = member
    return jsonify(member.to_dict()), 201


@app.route('/members', methods=['GET'])
@token_required
def get_members():
    page = int(request.args.get('page', 1))
    members = paginate(members_db, page, MEMBERS_PER_PAGE)
    return jsonify([member.to_dict() for member in members])


@app.route('/members/<int:id>', methods=['GET'])
@token_required
def get_member(id):
    member = members_db.get(id)
    if not member:
        return jsonify({"message": "Member not found"}), 404
    return jsonify(member.to_dict())


@app.route('/members/<int:id>', methods=['PUT'])
@token_required
def update_member(id):
    data = request.json
    member = members_db.get(id)
    if not member:
        return jsonify({"message": "Member not found"}), 404
    member.name = data['name']
    member.email = data['email']
    return jsonify(member.to_dict())


@app.route('/members/<int:id>', methods=['DELETE'])
@token_required
def delete_member(id):
    member = members_db.pop(id, None)
    if not member:
        return jsonify({"message": "Member not found"}), 404
    return jsonify({"message": "Member deleted"}), 200


# Run the app
if __name__ == '__main__':
    app.run(debug=True)
