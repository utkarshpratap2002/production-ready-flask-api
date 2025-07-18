# Production-Ready Flask API: A DevSecOps & SRE Case Study

This project is more than just a simple Flask CRUD API. It serves as a practical case study in diagnosing and solving the common challenges faced when moving an application from a local development environment to a containerized, production-like setting.

The core of the project is a simple "Book" API, but its true purpose is to demonstrate a journey of iterative improvement, tackling real-world deployment issues head-on. It showcases a thought process focused on creating robust, deployable, and maintainable services—key tenets of Site Reliability Engineering (SRE) and DevOps.

## Key Features

*   **Containerized with Docker:** A production-grade `Dockerfile` ensures predictable, isolated deployments.
*   **Production-Grade WSGI Server:** Uses Gunicorn for handling concurrent requests efficiently.
*   **Interactive API Documentation:** Integrated with Swagger UI for easy API exploration and testing.
*   **Environment-based Configuration:** Follows 12-Factor App principles by managing configuration (host, port, prefix) through environment variables (`.env` file).
*   **Dynamic Swagger Configuration:** The application dynamically updates its Swagger configuration on startup to reflect the correct server environment, preventing configuration drift.
*   **Robust Error Handling:** Implements custom handlers for common HTTP errors like 404 (Not Found) and 405 (Method Not Allowed).

## A Journey Through Production Hurdles: From `dev` to `docker`

Building this API involved solving a series of classic deployment problems. This narrative highlights the diagnostic and problem-solving process.

### Problem 1: The Boot Failure (`ModuleNotFoundError`)

*   **Symptom:** Gunicorn failed to boot the application with a `ModuleNotFoundError: No module named 'util.common'`.
*   **Root Cause:** A simple typo in a filename (`comman.py` instead of `common.py`) and missing `__init__.py` files, which prevented Python from recognizing the `util` and `resources` directories as importable packages.
*   **Solution:**
    1.  Corrected the filename typo.
    2.  Added empty `__init__.py` files to the `resources/` and `util/` directories to explicitly define them as Python packages.
*   **Lesson:** Reinforces the importance of standardized project structure and the value of linters, which would have caught this error before runtime.

### Problem 2: The CORS Conundrum in Docker (`Fetch error`)

*   **Symptom:** After containerizing the app, the Swagger UI page would load, but it failed to fetch its configuration file, showing a CORS error in the browser console. The page at `http://localhost:8080` was trying to fetch a resource from `http://localhost:5000`.
*   **Root Cause:** The application was generating an *absolute* URL for the Swagger configuration based on its internal port (`5000`). The browser's Same-Origin Policy correctly blocked this cross-port request.
*   **Solution:** The `application.py` was modified to provide a **relative URL** (`/api/v1/swagger-config`) to the Swagger UI blueprint. The browser now correctly requests the config from the same origin it loaded the page from (`http://localhost:8080`), which Docker then forwards to the application.
*   **Lesson:** Designing services to be environment-agnostic is crucial for portability. Hardcoding hostnames or ports creates brittle systems that fail behind proxies or in complex network environments.

### Problem 3: The Endpoint Collision (`AssertionError`)

*   **Symptom:** Gunicorn workers would start, then immediately crash with `AssertionError: View function mapping is overwriting an existing endpoint function: swaggerconfig`.
*   **Root Cause:** The `api.add_resource(SwaggerConfig, '/swagger-config')` line was accidentally duplicated in `application.py`. Flask-RESTful automatically generates an internal endpoint name from the class name (`SwaggerConfig` -> `swaggerconfig`), and Flask prohibits two routes from sharing the same internal name.
*   **Solution:** Removed the redundant `api.add_resource` call.
*   **Lesson:** Code must be clean and non-redundant. This also highlights the value of being explicit with endpoint naming (`endpoint='...'`) in larger applications to prevent accidental collisions.

## How to Run This Project

### Prerequisites

*   Git
*   Docker & Docker Compose

### Running with Docker (Recommended Method)

This is the simplest and most reliable way to run the application, as it mirrors a production deployment.

1.  **Clone the repository:**
    ```bash
    git clone <your-repo-url>
    cd <your-repo-name>
    ```

2.  **Create your environment file:**
    Copy the example and customize if needed.
    ```bash
    cp .env.example .env
    ```
    *(Note: You will need to create a `.env.example` file for your users)*

3.  **Build the Docker image:**
    ```bash
    docker build -t flask-api-project .
    ```

4.  **Run the container:**
    This command maps port 8080 on your local machine to port 5000 inside the container.
    ```bash
    docker run --rm -it -p 8080:5000 --name api-flask-container flask-api-project
    ```

5.  **Access the API:**
    *   **Swagger UI:** Open your browser to `http://localhost:8080/api/v1`
    *   **Get all books:** `curl http://localhost:8080/api/v1/books`

## API Endpoints

| Method | Path               | Description                 |
| :----- | :----------------- | :-------------------------- |
| `GET`  | `/api/v1/books`    | Retrieve all books          |
| `POST` | `/api/v1/books`    | Create a new book           |
| `GET`  | `/api/v1/books/id` | Retrieve a specific book    |
| `PUT`  | `/api/v1/books/id` | Update a specific book      |
| `DELETE`| `/api/v1/books/id`| Delete a specific book      |

## Areas for Future Improvement (Roadmap)

This project provides a solid foundation. The next steps to elevate it to a truly enterprise-grade service would include:

*   **Refactor API Resources:** Consolidate the separate `Book*Resource` classes for each HTTP method into a single `BookListResource` (for `/books`) and `BookResource` (for `/books/<id>`), which is a more idiomatic use of Flask-RESTful.
*   **Input Validation:** Implement robust input validation using a library like `Pydantic` or `Marshmallow` instead of `json.loads(request.data)` to prevent malformed data and enhance security.
*   **Dockerfile Hardening:**
    *   Implement a `.dockerignore` file to prevent copying `.venv`, `.git`, etc., into the image.
    *   Run the application as a non-root user for improved security.
    *   Use multi-stage builds to create a smaller final production image.
*   **Statelessness:** Replace the in-memory `books` list with a proper database (e.g., PostgreSQL, connected via SQLAlchemy) to persist data and allow for horizontal scaling.
*   **CI/CD Pipeline:** Set up a GitHub Actions workflow to automatically lint, test, and build the Docker image on every push.
*   **Observability:** Integrate structured logging (e.g., JSON logs) and add a `/metrics` endpoint for Prometheus scraping.