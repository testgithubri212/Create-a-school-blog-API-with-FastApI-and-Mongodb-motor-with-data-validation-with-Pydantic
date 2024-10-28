# Create-a-school-blog-API-with-FastApI-and-Mongodb-motor-with-data-validation-with-Pydantic
# school_blog_api.py

from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel, Field
from typing import Optional, List
from motor.motor_asyncio import AsyncIOMotorClient
from bson import ObjectId
from datetime import datetime
import os
from dotenv import load_dotenv

# Load environment variables from .env file
load_dotenv()

# Constants and Database Configuration
MONGODB_URL = os.getenv("MONGODB_URL", "mongodb://localhost:27017")
DB_NAME = os.getenv("DB_NAME", "school_blog")

client = AsyncIOMotorClient(MONGODB_URL)
database = client[DB_NAME]
blog_collection = database.get_collection("blog_posts")

# Helper to Convert MongoDB Document to Dictionary
def blog_post_helper(blog_post) -> dict:
    return {
        "id": str(blog_post["_id"]),
        "title": blog_post["title"],
        "content": blog_post["content"],
        "author": blog_post["author"],
        "created_at": blog_post["created_at"]
    }

# Data Models
class BlogPost(BaseModel):
    title: str = Field(..., max_length=100)
    content: str = Field(..., min_length=10)
    author: str = Field(..., max_length=50)
    created_at: Optional[datetime] = Field(default_factory=datetime.utcnow)

class BlogPostUpdate(BaseModel):
    title: Optional[str] = Field(None, max_length=100)
    content: Optional[str] = Field(None, min_length=10)
    author: Optional[str] = Field(None, max_length=50)

class BlogPostResponse(BlogPost):
    id: str

# CRUD Operations
async def create_blog_post(blog_post: BlogPost) -> dict:
    post = await blog_collection.insert_one(blog_post.dict())
    new_post = await blog_collection.find_one({"_id": post.inserted_id})
    return blog_post_helper(new_post)

async def get_blog_posts() -> list:
    posts = []
    async for post in blog_collection.find():
        posts.append(blog_post_helper(post))
    return posts

async def get_blog_post(id: str) -> dict:
    post = await blog_collection.find_one({"_id": ObjectId(id)})
    if post:
        return blog_post_helper(post)

async def update_blog_post(id: str, data: BlogPostUpdate):
    update_result = await blog_collection.update_one(
        {"_id": ObjectId(id)}, {"$set": data.dict(exclude_unset=True)}
    )
    if update_result.modified_count:
        updated_post = await blog_collection.find_one({"_id": ObjectId(id)})
        return blog_post_helper(updated_post)

async def delete_blog_post(id: str):
    delete_result = await blog_collection.delete_one({"_id": ObjectId(id)})
    return delete_result.deleted_count > 0

# FastAPI Application
app = FastAPI()

@app.post("/posts/", response_model=BlogPostResponse)
async def create_post(blog_post: BlogPost):
    post = await create_blog_post(blog_post)
    return post

@app.get("/posts/", response_model=List[BlogPostResponse])
async def read_posts():
    return await get_blog_posts()

@app.get("/posts/{post_id}", response_model=BlogPostResponse)
async def read_post(post_id: str):
    post = await get_blog_post(post_id)
    if not post:
        raise HTTPException(status_code=404, detail="Post not found")
    return post

@app.put("/posts/{post_id}", response_model=BlogPostResponse)
async def update_post(post_id: str, blog_post: BlogPostUpdate):
    updated_post = await update_blog_post(post_id, blog_post)
    if not updated_post:
        raise HTTPException(status_code=404, detail="Post not found")
    return updated_post

@app.delete("/posts/{post_id}")
async def delete_post(post_id: str):
    deleted = await delete_blog_post(post_id)
    if not deleted:
        raise HTTPException(status_code=404, detail="Post not found")
    return {"message": "Post deleted successfully"}
    
    #Usage Instructions
1. Install Required Packages:

pip install fastapi motor pydantic uvicorn python-dotenv


2. Create a .env File:

In the same directory, create a .env file with your MongoDB URL:

MONGODB_URL=mongodb://localhost:27017
DB_NAME=school_blog


3. Run the Application:

Run the API with Uvicorn:

uvicorn school_blog_api:app --reload


4. Access API Documentation:

Go to http://127.0.0.1:8000/docs to view and test the API using FastAPI's Swagger UI.




---

Example Requests and Responses:

POST /posts/ (Create a new blog post)

Request:

{
  "title": "Introduction to FastAPI",
  "content": "FastAPI is a modern web framework for building APIs with Python...",
  "author": "Sai Priya"
}

Response:

{
  "id": "6123456789abcdef01234567",
  "title": "Introduction to FastAPI",
  "content": "FastAPI is a modern web framework for building APIs with Python...",
  "author": "Sai Priya",
  "created_at": "2024-10-28T12:34:56.789Z"
}

GET /posts/ (Retrieve all blog posts)

Response:

[
  
    "id": "6123456789abcdef01234567",
    "title": "Introduction to FastAPI",
    "content": "FastAPI is a modern web framework for building APIs with Python...",
    "author": "Sai Priya",
    "created_at": "2024-10-28T12:34:56.789Z"
  }
]

DELETE /posts/{post_id} (Delete a blog post)

Response:

{
  "message": "Post deleted successfully"
}



