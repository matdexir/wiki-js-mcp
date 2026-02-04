# AGENTS.md - AI Agent Guidelines for Wiki.js MCP Server

This file contains essential guidelines for AI agents working on the Wiki.js MCP server codebase. Follow these conventions to maintain consistency and code quality.

## Build, Lint, and Test Commands

### Setup and Installation
```bash
# Initial environment setup
./setup.sh

# Install dependencies manually (if setup fails)
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### Code Quality and Formatting
```bash
# Format code with Black (line length: 88)
black src/ --line-length 88

# Sort imports with isort (profile: black)
isort src/ --profile black

# Type checking with mypy (Python 3.12)
mypy src/wiki_mcp_server.py --python-version 3.12

# Run all formatting/linting in sequence
black src/ && isort src/ && mypy src/
```

### Testing and Development
```bash
# Test server connection and configuration
./test-server.sh

# Start MCP server for development/testing
./start-server.sh

# Run with debug logging
LOG_LEVEL=DEBUG ./start-server.sh
```

### Single Test Execution
No formal test suite exists. Use the test-server.sh script for manual testing:
```bash
# Interactive testing with validation
./test-server.sh
```

## Code Style Guidelines

### Python Conventions
- **Python Version**: 3.12+ required
- **Style Guide**: Follow PEP 8 with Black formatting
- **Line Length**: 88 characters (configured in pyproject.toml)
- **Import Organization**: Use isort with black profile
- **Type Hints**: Required for all function signatures (mypy strict mode)

### Import Structure
```python
# Standard library imports first
import os
import sys
from datetime import datetime
from pathlib import Path
from typing import Optional, List, Dict, Any, Union

# Third-party imports next
import httpx
from fastmcp import FastMCP
from pydantic import Field
from pydantic_settings import BaseSettings
from sqlalchemy import create_engine, Column, Integer, String
from tenacity import retry, stop_after_attempt, wait_exponential

# Local imports last
from .models import FileMapping, RepositoryContext
```

### Naming Conventions
- **Classes**: PascalCase (e.g., `FileMapping`, `RepositoryContext`)
- **Functions/Methods**: snake_case (e.g., `create_page`, `get_page_children`)
- **Variables**: snake_case (e.g., `page_id`, `repository_root`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `DEFAULT_SPACE_NAME`, `MAX_RETRY_ATTEMPTS`)
- **Private methods**: Leading underscore (e.g., `_validate_token()`, `_execute_graphql()`)

### Type Hints and Validation
```python
from typing import Optional, List, Dict, Any, Union
from pydantic import Field
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    WIKIJS_API_URL: str = Field(default="http://localhost:3000")
    WIKIJS_TOKEN: Optional[str] = Field(default=None)
    
    @property
    def token(self) -> Optional[str]:
        """Get the token from either WIKIJS_TOKEN or WIKIJS_API_KEY."""
        return self.WIKIJS_TOKEN or self.WIKIJS_API_KEY

async def create_page(
    title: str,
    content: str,
    parent_id: Optional[int] = None
) -> Dict[str, Any]:
    """Create a new page in Wiki.js."""
    pass
```

### Error Handling
```python
import logging
from tenacity import retry, stop_after_attempt, wait_exponential
from sqlalchemy.exc import SQLAlchemyError

logger = logging.getLogger(__name__)

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
async def _execute_graphql(query: str, variables: Dict[str, Any]) -> Dict[str, Any]:
    """Execute GraphQL query with retry logic."""
    try:
        response = await self.http_client.post(
            f"{self.settings.WIKIJS_API_URL}/graphql",
            json={"query": query, "variables": variables}
        )
        response.raise_for_status()
        return response.json()
    except httpx.HTTPStatusError as e:
        logger.error(f"GraphQL request failed: {e}")
        raise
    except Exception as e:
        logger.error(f"Unexpected error in GraphQL request: {e}")
        raise

try:
    result = await _execute_graphql(query, variables)
except SQLAlchemyError as e:
    logger.error(f"Database error: {e}")
    raise ValueError("Failed to save page mapping") from e
```

### Configuration Management
- Use Pydantic Settings for configuration
- Support both .env file and environment variables
- Provide sensible defaults
- Validate required fields on startup

```python
class Settings(BaseSettings):
    class Config:
        env_file = ".env"
        extra = "ignore"  # Allow extra fields without validation errors
```

### Database Patterns
- Use SQLAlchemy ORM with declarative base
- Include timestamp columns (created_at, updated_at)
- Use context managers for sessions
- Handle SQLAlchemy exceptions gracefully

```python
from sqlalchemy import create_engine, Column, Integer, String, DateTime
from sqlalchemy.orm import declarative_base, sessionmaker
from datetime import datetime

Base = declarative_base()

class FileMapping(Base):
    __tablename__ = "file_mappings"
    
    id = Column(Integer, primary_key=True)
    file_path = Column(String, unique=True, nullable=False)
    page_id = Column(Integer, nullable=False)
    last_updated = Column(DateTime, default=datetime.utcnow)
```

### MCP Tool Implementation
- Use FastMCP decorators for tool registration
- Include comprehensive docstrings
- Use async/await for all I/O operations
- Return structured dictionaries as results
- Handle errors and return meaningful error messages

```python
@mcp.tool()
async def create_page(
    title: str,
    content: str,
    parent_id: Optional[int] = None
) -> Dict[str, Any]:
    """Create a new page in Wiki.js.
    
    Args:
        title: Page title
        content: Page content in markdown format
        parent_id: Optional parent page ID for hierarchy
        
    Returns:
        Dictionary containing page_id, title, and path
    """
    try:
        # Implementation here
        return {"page_id": page_id, "title": title, "path": path}
    except Exception as e:
        return {"error": str(e), "success": False}
```

### Logging Standards
- Use Python's logging module
- Include context in log messages
- Use appropriate log levels (DEBUG, INFO, WARNING, ERROR)
- Log both file and console handlers

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    handlers=[logging.FileHandler("app.log"), logging.StreamHandler()]
)
logger = logging.getLogger(__name__)

logger.info("Starting MCP server")
logger.debug(f"Processing request: {request_data}")
logger.error(f"Failed to create page: {error}")
```

### Documentation and Comments
- Use docstrings for all public functions and classes
- Include type hints in function signatures
- Comment complex business logic
- Keep comments concise and up-to-date

### File Organization
```
src/
├── wiki_mcp_server.py    # Main MCP server with all tools
config/
├── example.env           # Configuration template
```

### Environment Variables
Required environment variables in .env:
```bash
# Wiki.js Connection
WIKIJS_API_URL=http://localhost:3000
WIKIJS_TOKEN=your_jwt_token_here
# OR
WIKIJS_USERNAME=your_username
WIKIJS_PASSWORD=your_password

# Database & Logging
WIKIJS_MCP_DB=./wikijs_mappings.db
LOG_LEVEL=INFO
LOG_FILE=wikijs_mcp.log
```

### GraphQL Integration
- Use httpx for async HTTP requests
- Implement retry logic with tenacity
- Handle GraphQL-specific errors
- Use query variables for parameterization

### Security Best Practices
- Never log sensitive data (tokens, passwords)
- Validate all inputs
- Use environment variables for secrets
- Implement proper error handling without information disclosure

### Performance Guidelines
- Use async/await for all I/O operations
- Implement connection pooling for HTTP clients
- Cache frequently accessed data
- Use batch operations for bulk updates

## Repository Context

This is a Model Context Protocol (MCP) server that provides Wiki.js integration with hierarchical documentation support. It uses:
- **FastMCP**: Python MCP SDK
- **SQLAlchemy**: ORM for local SQLite database
- **httpx**: Async HTTP client for GraphQL
- **Pydantic**: Settings and validation
- **tenacity**: Retry logic

The server provides 21 tools for documentation management, hierarchical page creation, and file-to-page mapping.