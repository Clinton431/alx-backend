"""
Detailed examples of API and Database Pagination in Python
"""

# ======================================================================
# PART 1: DATABASE PAGINATION
# ======================================================================

"""
Database pagination is crucial when working with large datasets in databases.
Fetching all records at once can be memory-intensive and slow. Pagination
allows retrieving only the records needed for the current view.
"""

# Example with SQLAlchemy ORM
def sqlalchemy_pagination_example():
    from sqlalchemy import create_engine, Column, Integer, String, func
    from sqlalchemy.ext.declarative import declarative_base
    from sqlalchemy.orm import sessionmaker
    
    # Set up SQLAlchemy
    Base = declarative_base()
    engine = create_engine('sqlite:///:memory:')
    Session = sessionmaker(bind=engine)
    
    # Define a simple model
    class Product(Base):
        __tablename__ = 'products'
        id = Column(Integer, primary_key=True)
        name = Column(String)
        price = Column(Integer)  # Price in cents
        
        def __repr__(self):
            return f"Product({self.id}, {self.name}, ${self.price/100:.2f})"
    
    # Create tables and sample data
    Base.metadata.create_all(engine)
    session = Session()
    
    # Insert 100 sample products
    for i in range(1, 101):
        price = i * 100  # $1.00 to $100.00
        session.add(Product(name=f"Product {i}", price=price))
    session.commit()
    
    print("Database Pagination with SQLAlchemy\n")
    
    # ----------------------------------------------------------------------
    # Method 1: Basic LIMIT and OFFSET pagination
    # ----------------------------------------------------------------------
    print("Method 1: Basic LIMIT and OFFSET pagination")
    
    page = 2
    per_page = 10
    offset = (page - 1) * per_page
    
    # Count total records for pagination metadata
    total_records = session.query(Product).count()
    total_pages = (total_records + per_page - 1) // per_page
    
    # Get paginated results
    products = session.query(Product).order_by(Product.id).offset(offset).limit(per_page).all()
    
    print(f"Page {page} of {total_pages} (Total records: {total_records})")
    for product in products:
        print(f"  {product}")
    
    # ----------------------------------------------------------------------
    # Method 2: Using SQLAlchemy's Pagination through a custom function
    # ----------------------------------------------------------------------
    print("\nMethod 2: SQLAlchemy Pagination with Custom Function")
    
    def paginate(query, page, per_page):
        # Get total count once
        total = query.count()
        # Calculate pagination values
        total_pages = (total + per_page - 1) // per_page
        page = min(page, total_pages) if total_pages > 0 else 1
        page = max(page, 1)  # Ensure page is at least 1
        offset = (page - 1) * per_page
        # Execute query with limit/offset
        items = query.offset(offset).limit(per_page).all()
        
        # Return pagination data
        return {
            'items': items,
            'page': page,
            'per_page': per_page, 
            'total': total,
            'total_pages': total_pages,
            'has_next': page < total_pages,
            'has_prev': page > 1
        }
    
    # Create a base query that can include filters
    query = session.query(Product).filter(Product.price > 1500).order_by(Product.price.desc())
    
    # Get page 1 with 5 items per page
    result = paginate(query, page=1, per_page=5)
    
    print(f"Page {result['page']} of {result['total_pages']} (Filtered records: {result['total']})")
    print(f"Has next page: {result['has_next']}, Has previous page: {result['has_prev']}")
    for product in result['items']:
        print(f"  {product}")
    
    # ----------------------------------------------------------------------
    # Method 3: Keyset Pagination (more efficient for large datasets)
    # ----------------------------------------------------------------------
    print("\nMethod 3: Keyset Pagination (cursor-based)")
    
    # Initial query - get first page
    page_size = 5
    first_page = session.query(Product).order_by(Product.id).limit(page_size).all()
    
    print("First page:")
    for product in first_page:
        print(f"  {product}")
    
    if first_page:
        # Get the last ID from the current page to use as cursor
        last_id = first_page[-1].id
        
        # Get next page using the cursor (more efficient than offset)
        next_page = session.query(Product).filter(Product.id > last_id).order_by(Product.id).limit(page_size).all()
        
        print("\nNext page (using keyset/cursor):")
        for product in next_page:
            print(f"  {product}")
    
    # ----------------------------------------------------------------------
    # Method 4: Window Functions (available in PostgreSQL, SQLite 3.25+)
    # ----------------------------------------------------------------------
    print("\nMethod 4: Window Functions (advanced pagination)")
    # Note: This would work in PostgreSQL but may not work in all SQLite versions
    
    try:
        # Query with row numbers to get specific page
        page = 3
        per_page = 5
        
        # This syntax works in PostgreSQL
        # In SQLite 3.25+ you'd use ROW_NUMBER() OVER()
        # The below is conceptual and may need adjustments for specific databases
        stmt = """
        SELECT * FROM (
            SELECT *, ROW_NUMBER() OVER (ORDER BY id) as row_num 
            FROM products
        ) AS numbered
        WHERE row_num BETWEEN :start AND :end
        """
        
        start = (page - 1) * per_page + 1
        end = start + per_page - 1
        
        # Try executing - this may fail depending on SQLite version
        products = session.execute(stmt, {"start": start, "end": end}).fetchall()
        
        print(f"Page {page} (rows {start}-{end}):")
        for product in products:
            print(f"  Product({product.id}, {product.name}, ${product.price/100:.2f})")
    
    except Exception as e:
        print(f"Window function example failed (may not be supported in your SQLite version): {e}")
    
    session.close()


# Example with Django-style pagination
def django_style_pagination():
    """
    Django's ORM has built-in pagination which is very convenient.
    This example mimics Django's Paginator class functionality.
    """
    
    class Paginator:
        def __init__(self, object_list, per_page):
            """Initialize the paginator with a list of objects and page size."""
            self.object_list = object_list
            self.per_page = per_page
            self.count = len(object_list)
            self.num_pages = (self.count + per_page - 1) // per_page
        
        def page(self, number):
            """Return a Page object for the given page number."""
            number = max(1, min(number, self.num_pages))
            bottom = (number - 1) * self.per_page
            top = bottom + self.per_page
            return self.object_list[bottom:top]
    
    # Create sample data
    products = [f"Product {i}" for i in range(1, 101)]
    
    # Create paginator with 10 items per page
    paginator = Paginator(products, per_page=10)
    
    # Get page 3
    page_num = 3
    page_obj = paginator.page(page_num)
    
    print("\nDjango-style Pagination:")
    print(f"Page {page_num} of {paginator.num_pages}")
    print(f"Items: {page_obj}")


# ======================================================================
# PART 2: API PAGINATION
# ======================================================================

"""
API pagination involves handling paginated responses from APIs or
implementing pagination in your own API endpoints. There are several
common patterns for API pagination.
"""

def api_pagination_patterns():
    import requests
    import json
    from urllib.parse import urlencode
    
    print("\n\nAPI PAGINATION PATTERNS\n")
    
    # ----------------------------------------------------------------------
    # Pattern 1: Page-based pagination (GitHub-style)
    # ----------------------------------------------------------------------
    print("Pattern 1: Page-based pagination (GitHub-style)")
    
    def fetch_github_repos(page=1, per_page=5):
        """Fetch public repositories from GitHub API with page-based pagination."""
        base_url = "https://api.github.com/repositories"
        params = {
            "page": page,
            "per_page": per_page
        }
        
        # Mock the API call (remove this in real code)
        print(f"GET {base_url}?{urlencode(params)}")
        
        # In a real application, you would do:
        # response = requests.get(base_url, params=params)
        # return response.json(), response.headers
        
        # For demonstration, we'll simulate the response:
        fake_repos = [
            {"id": f"{page}0{i}", "name": f"repo-{page}-{i}"} 
            for i in range(1, per_page + 1)
        ]
        fake_headers = {
            "Link": f'<{base_url}?page={page+1}&per_page={per_page}>; rel="next", '
                   f'<{base_url}?page={10}&per_page={per_page}>; rel="last"'
        }
        return fake_repos, fake_headers
    
    # Fetch first page
    repos, headers = fetch_github_repos(page=1)
    
    print("First page results:")
    for repo in repos:
        print(f"  {repo['name']} (ID: {repo['id']})")
    
    # Parse Link header to get next page URL
    link_header = headers.get('Link', '')
    next_url = None
    
    if 'rel="next"' in link_header:
        # In a real implementation, you would parse the URL properly
        print("\nNext page available via Link header")
        
        # Fetch second page
        repos, headers = fetch_github_repos(page=2)
        
        print("\nSecond page results:")
        for repo in repos:
            print(f"  {repo['name']} (ID: {repo['id']})")
    
    # ----------------------------------------------------------------------
    # Pattern 2: Offset-based pagination
    # ----------------------------------------------------------------------
    print("\nPattern 2: Offset-based pagination")
    
    def fetch_with_offset(offset=0, limit=5):
        """Fetch data using offset-based pagination."""
        base_url = "https://api.example.com/items"
        params = {
            "offset": offset,
            "limit": limit
        }
        
        # Mock the API call
        print(f"GET {base_url}?{urlencode(params)}")
        
        # Simulate response
        fake_items = [
            {"id": i, "name": f"Item {i}"} 
            for i in range(offset, offset + limit)
        ]
        fake_metadata = {
            "total": 100,
            "offset": offset,
            "limit": limit
        }
        
        return {
            "items": fake_items,
            "meta": fake_metadata
        }
    
    # Fetch first page
    response = fetch_with_offset(offset=0, limit=5)
    
    print("First page results:")
    for item in response["items"]:
        print(f"  {item['name']} (ID: {item['id']})")
    
    # Calculate next offset
    next_offset = response["meta"]["offset"] + response["meta"]["limit"]
    
    # Fetch next page
    if next_offset < response["meta"]["total"]:
        response = fetch_with_offset(offset=next_offset, limit=5)
        
        print("\nSecond page results:")
        for item in response["items"]:
            print(f"  {item['name']} (ID: {item['id']})")
    
    # ----------------------------------------------------------------------
    # Pattern 3: Cursor-based pagination (Facebook/Twitter style)
    # ----------------------------------------------------------------------
    print("\nPattern 3: Cursor-based pagination")
    
    def fetch_with_cursor(cursor=None, limit=5):
        """Fetch data using cursor-based pagination."""
        base_url = "https://api.example.com/feed"
        params = {"limit": limit}
        
        if cursor:
            params["cursor"] = cursor
        
        # Mock the API call
        print(f"GET {base_url}?{urlencode(params)}")
        
        # Simulate response with a cursor
        start_id = 100 if cursor is None else int(cursor)
        fake_posts = [
            {"id": start_id - i, "content": f"Post content {start_id - i}"} 
            for i in range(limit)
        ]
        
        # Provide next cursor if more data exists
        next_cursor = str(start_id - limit) if start_id > limit else None
        
        return {
            "data": fake_posts,
            "paging": {
                "cursors": {
                    "next": next_cursor
                },
                "has_next_page": next_cursor is not None
            }
        }
    
    # Fetch first page (no cursor)
    response = fetch_with_cursor()
    
    print("First page results:")
    for post in response["data"]:
        print(f"  Post {post['id']}: {post['content']}")
    
    # Check if there's a next page
    if response["paging"]["has_next_page"]:
        next_cursor = response["paging"]["cursors"]["next"]
        
        # Fetch next page using the cursor
        response = fetch_with_cursor(cursor=next_cursor)
        
        print("\nSecond page results (using cursor):")
        for post in response["data"]:
            print(f"  Post {post['id']}: {post['content']}")
    
    # ----------------------------------------------------------------------
    # Pattern 4: Page token pagination (Google APIs style)
    # ----------------------------------------------------------------------
    print("\nPattern 4: Page token pagination (Google style)")
    
    def fetch_with_page_token(page_token=None, page_size=5):
        """Fetch data using page token pagination (Google APIs style)."""
        base_url = "https://api.example.com/resources"
        params = {"pageSize": page_size}
        
        if page_token:
            params["pageToken"] = page_token
        
        # Mock the API call
        print(f"GET {base_url}?{urlencode(params)}")
        
        # Simulate response with a page token
        start_id = 1 if page_token is None else int(page_token)
        fake_resources = [
            {"id": f"res-{start_id + i}", "name": f"Resource {start_id + i}"} 
            for i in range(page_size)
        ]
        
        # Provide next page token if more data exists
        next_page_token = str(start_id + page_size) if start_id + page_size <= 50 else None
        
        return {
            "resources": fake_resources,
            "nextPageToken": next_page_token
        }
    
    # Fetch first page
    response = fetch_with_page_token()
    
    print("First page results:")
    for resource in response["resources"]:
        print(f"  {resource['name']} (ID: {resource['id']})")
    
    # Check if there's a next page token
    if response.get("nextPageToken"):
        next_token = response["nextPageToken"]
        
        # Fetch next page using the token
        response = fetch_with_page_token(page_token=next_token)
        
        print("\nSecond page results (using page token):")
        for resource in response["resources"]:
            print(f"  {resource['name']} (ID: {resource['id']})")


# ======================================================================
# PART 3: IMPLEMENTING PAGINATION IN YOUR OWN API
# ======================================================================

"""
When building APIs, you'll need to implement pagination on the server side.
Here's how you might do it with Flask and SQLAlchemy.
"""

def flask_api_pagination_example():
    """
    Example of implementing pagination in a Flask API.
    This is just a code example and won't actually run.
    """
    from flask import Flask, request, jsonify
    from flask_sqlalchemy import SQLAlchemy
    
    # Set up Flask and SQLAlchemy (this is just for demonstration)
    app = Flask(__name__)
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///example.db'
    db = SQLAlchemy(app)
    
    # Define a model
    class User(db.Model):
        id = db.Column(db.Integer, primary_key=True)
        username = db.Column(db.String(80), unique=True, nullable=False)
        email = db.Column(db.String(120), unique=True, nullable=False)
    
    # API endpoint with pagination
    @app.route('/api/users', methods=['GET'])
    def get_users():
        # Get pagination parameters from request
        page = request.args.get('page', 1, type=int)
        per_page = min(request.args.get('per_page', 10, type=int), 100)  # Limit max items
        
        # Query with pagination
        query = User.query.order_by(User.id)
        paginated_users = query.paginate(page=page, per_page=per_page, error_out=False)
        
        # Prepare response
        response = {
            'users': [{'id': user.id, 'username': user.username} for user in paginated_users.items],
            'pagination': {
                'page': page,
                'per_page': per_page,
                'total': paginated_users.total,
                'pages': paginated_users.pages,
                'has_next': paginated_users.has_next,
                'has_prev': paginated_users.has_prev
            }
        }
        
        # Add navigation links
        if paginated_users.has_next:
            response['pagination']['next_url'] = f'/api/users?page={page+1}&per_page={per_page}'
        if paginated_users.has_prev:
            response['pagination']['prev_url'] = f'/api/users?page={page-1}&per_page={per_page}'
        
        return jsonify(response)
    
    print("\nFlask API Pagination Example (Code Only)")
    print("This demonstrates how to implement pagination in a Flask API.")


# ======================================================================
# PART 4: FAST API PAGINATION EXAMPLE
# ======================================================================

def fastapi_pagination_example():
    """
    Example of implementing pagination in a FastAPI application.
    This is just a code example and won't actually run.
    """
    print("\nFastAPI Pagination Example (Code Only)")
    
    # This code would be in a real application
    code_example = """
    from fastapi import FastAPI, Depends, Query
    from sqlalchemy.orm import Session
    from typing import List, Optional
    from pydantic import BaseModel
    
    app = FastAPI()
    
    # Pydantic models
    class UserBase(BaseModel):
        username: str
        email: str
        
    class User(UserBase):
        id: int
        
        class Config:
            orm_mode = True
    
    class PaginatedResponse(BaseModel):
        items: List[User]
        total: int
        page: int
        size: int
        pages: int
        
    # Example endpoint
    @app.get("/users/", response_model=PaginatedResponse)
    def read_users(
        db: Session = Depends(get_db),
        skip: int = Query(0, ge=0),
        limit: int = Query(10, ge=1, le=100)
    ):
        # Calculate page number from skip/limit
        page = skip // limit + 1 if limit > 0 else 1
        
        # Get total count
        total = db.query(UserModel).count()
        
        # Get paginated items
        users = db.query(UserModel).offset(skip).limit(limit).all()
        
        # Calculate total pages
        pages = (total + limit - 1) // limit if limit > 0 else 1
        
        return {
            "items": users,
            "total": total,
            "page": page,
            "size": limit,
            "pages": pages
        }
    """
    
    print("FastAPI makes it easy to implement pagination with query parameters.")
    print("The example shows how to create a paginated endpoint with skip/limit parameters.")


# Run the examples (comment out as needed)
if __name__ == "__main__":
    print("=" * 80)
    print("DATABASE PAGINATION EXAMPLES")
    print("=" * 80)
    sqlalchemy_pagination_example()
    #django_style_pagination()
    
    print("\n" + "=" * 80)
    print("API PAGINATION EXAMPLES")
    print("=" * 80)
    api_pagination_patterns()
    #flask_api_pagination_example()
    #fastapi_pagination_example()
