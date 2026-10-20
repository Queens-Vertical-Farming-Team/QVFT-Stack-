# Contributing to QVFT Stack

Thank you for contributing to the Queen's Vertical Farming Team Stack project.

This repository is maintained by the QVFT Stack Team and is intended to support collaborative, open-source development of the software and automation systems used by the farm.

## Development Workflow

Do not commit directly to `main` once the repository setup is complete.

Instead, use the following workflow:

1. Create or claim a GitHub Issue
2. Create a new branch
3. Make your changes
4. Test your changes
5. Open a Pull Request
6. Request a review
7. Merge only after approval

## Branch Naming

Use clear branch names based on the type of work being completed.

Examples:

```text
feature/dashboard
feature/sensor-api
feature/esp32-temperature
feature/database-schema

fix/device-timeout
fix/dashboard-loading

docs/backend-setup
docs/sensor-guide

Creating a Branch
Example:
git checkout -b feature/my-feature

Commit Messages
Use short, descriptive commit messages.

Examples:
feat: add temperature sensor endpoint
fix: correct humidity parsing
docs: update backend setup guide
chore: reorganize sample data

Recommended prefixes:
- feat: new functionality
- fix: bug fixes
- docs: documentation changes
- test: testing changes
- refactor: code restructuring
- chore: maintenance or setup changes

Pull Requests
Every Pull Request should explain:
- What was changed
- Why the change was needed
- How the change was tested
- Any known limitations
- Any related GitHub Issues
Keep Pull Requests focused on one feature, fix, or task whenever possible.
Code Review
At least one other QVFT Stack Team member should review significant changes before they are merged.

Reviewers should check:
- Does the code work?
- Is the code understandable?
- Does it introduce security risks?
- Does it follow the project structure?
- Is documentation needed?
- Are secrets or credentials exposed?
Security
Never commit:
- Passwords
- API keys
- Database credentials
- Private keys
- .env files
- Personal information
- Sensitive farm data

Use environment variables for sensitive configuration.

The repository should only contain:
.env.example

The real .env file must remain local.
Documentation
New systems or major changes should include documentation.

Relevant documentation should be added to:
docs/

Examples include:
- New API endpoints
- Sensor setup
- Database changes
- Device configuration
- Deployment steps
- Architecture changes
Testing
Contributors should test their changes before opening a Pull Request.

Testing may include:
- Running backend tests
- Testing API endpoints
- Verifying sensor readings
- Checking database writes
- Running the frontend locally
- Testing with sample data
Repository Areas
The main project areas are:
firmware/      Embedded systems and device code
backend/       API and server-side software
frontend/      Dashboard and user interface
database/      Database schema and migrations
docs/          Technical documentation
scripts/       Utility and testing scripts
sample-data/   Public test and demonstration data

Questions
If you are unsure where a change belongs or how it should be implemented, discuss it with the QVFT Stack Team before beginning major work.





