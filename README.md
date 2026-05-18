📚 Library Management System — Django REST API



<img width="1533" height="787" alt="WhatsApp Image 2026-05-18 at 10 48 09 AM" src="https://github.com/user-attachments/assets/d800ece8-d047-4720-8c6f-1335e942781e" />


🗂️ Project Structure (What We're Building)
library_api/
├── manage.py
├── library_api/          ← project config
│   ├── settings.py
│   ├── urls.py
├── books/                ← our app
│   ├── models.py
│   ├── serializers.py
│   ├── views.py
│   ├── urls.py
API Endpoints we'll create:



<img width="720" height="378" alt="api" src="https://github.com/user-attachments/assets/ce2f7aa1-c36d-4851-bbf1-d4a570e6e45d" />

MethodURLActionGET/api/books/List all booksPOST/api/books/Add a new bookGET/api/books/1/Get book by IDPUT/api/books/1/Update a bookDELETE/api/books/1/Delete a book

✅ Step 1 — Install Dependencies
bashpip install django djangorestframework

✅ Step 2 — Create the Project & App
bashdjango-admin startproject library_api
cd library_api
python manage.py startapp books

startapp — creates a modular (self-contained) Django application. Think of it like creating a .h + .c file pair in C, but for a feature.


✅ Step 3 — Register Apps in settings.py
library_api/settings.py
pythonINSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # Third-party
    'rest_framework',   # ← DRF

    # Our app
    'books',            # ← our books app
]

✅ Step 4 — Create the Model

Model = your database table, like a struct in C but with built-in DB powers.

books/models.py
pythonfrom django.db import models

class Book(models.Model):
    title       = models.CharField(max_length=200)
    author      = models.CharField(max_length=100)
    isbn        = models.CharField(max_length=13, unique=True)  # unique = no duplicates
    published   = models.DateField()
    available   = models.BooleanField(default=True)  # is the book available to borrow?

    def __str__(self):
        # Like printf representation — shown in admin panel
        return f"{self.title} by {self.author}"
Run migrations (creates the actual DB table):
bashpython manage.py makemigrations
python manage.py migrate

Migration — Django's way of translating your Python model into SQL CREATE TABLE statements. You don't write SQL manually!


✅ Step 5 — Create the Serializer

Serializer = converts Python objects ↔ JSON. Think of it as json.dumps() but with validation built in. It's the intermediary between your model and the API response.

books/serializers.py
pythonfrom rest_framework import serializers
from .models import Book

class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model  = Book
        fields = '__all__'  # include all fields: id, title, author, isbn, published, available
Example — what this does:
Book object  →  Serializer  →  JSON response
{ id:1, title:"1984", author:"Orwell" ... }

✅ Step 6 — Create the Views

View = handles the HTTP request and returns a response. Like a function in C that takes input and returns output.

books/views.py
pythonfrom rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework import status
from .models import Book
from .serializers import BookSerializer


# GET all books / POST a new book
@api_view(['GET', 'POST'])
def book_list(request):

    if request.method == 'GET':
        books = Book.objects.all()              # SELECT * FROM books
        serializer = BookSerializer(books, many=True)  # many=True → list of objects
        return Response(serializer.data)        # returns JSON

    elif request.method == 'POST':
        serializer = BookSerializer(data=request.data)  # request.data = JSON body sent by client
        if serializer.is_valid():               # validates fields (like null checks in C)
            serializer.save()                   # INSERT INTO books ...
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


# GET / PUT / DELETE a single book by id
@api_view(['GET', 'PUT', 'DELETE'])
def book_detail(request, pk):

    try:
        book = Book.objects.get(pk=pk)          # SELECT * FROM books WHERE id=pk
    except Book.DoesNotExist:
        return Response({'error': 'Book not found'}, status=status.HTTP_404_NOT_FOUND)

    if request.method == 'GET':
        serializer = BookSerializer(book)
        return Response(serializer.data)

    elif request.method == 'PUT':
        serializer = BookSerializer(book, data=request.data)  # update existing book
        if serializer.is_valid():
            serializer.save()                   # UPDATE books SET ... WHERE id=pk
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    elif request.method == 'DELETE':
        book.delete()                           # DELETE FROM books WHERE id=pk
        return Response(status=status.HTTP_204_NO_CONTENT)

✅ Step 7 — Set Up URLs
books/urls.py ← create this file
pythonfrom django.urls import path
from . import views

urlpatterns = [
    path('books/',      views.book_list,   name='book-list'),    # /api/books/
    path('books/<int:pk>/', views.book_detail, name='book-detail'),  # /api/books/1/
]
library_api/urls.py ← wire the app URLs into the project
pythonfrom django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('books.urls')),   # all books URLs are prefixed with /api/
]

✅ Step 8 — Run & Test
bashpython manage.py runserver
Test with curl (like making HTTP requests from terminal):
bash# GET all books
curl http://127.0.0.1:8000/api/books/

# POST a new book
curl -X POST http://127.0.0.1:8000/api/books/ \
  -H "Content-Type: application/json" \
  -d '{"title":"1984","author":"George Orwell","isbn":"1234567890123","published":"1949-06-08","available":true}'

# GET single book
curl http://127.0.0.1:8000/api/books/1/

# DELETE a book
curl -X DELETE http://127.0.0.1:8000/api/books/1/

🔄 Full Data Flow (Mental Model)
Client (curl/Postman/browser)
        │
        │  HTTP Request (JSON)
        ▼
    urls.py  ──→  routes to correct view function
        │
        ▼
    views.py  ──→  handles logic (GET/POST/PUT/DELETE)
        │
        ▼
  serializers.py  ──→  validates + converts data
        │
        ▼
    models.py  ──→  talks to the database (SQLite by default)
        │
        ▼
    Response (JSON) sent back to client

📌 Vocabulary Glossary
TermMeaningSerializationConverting complex objects (like DB rows) into a transmittable format (JSON). "Serialization" literally means converting to a series of bytes.EndpointA specific URL that accepts requests — analogous to a function entry pointMigrationA versioned script that evolves your DB schemaCRUDCreate, Read, Update, Delete — the four canonical (standard, universally accepted) database operationspkPrimary Key — unique identifier for each row, like an array index
