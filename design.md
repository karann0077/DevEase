# Design Document: AI for Bharat – Smart Documentation & Debugging Assistant

## Overview

The AI for Bharat Smart Documentation & Debugging Assistant is a microservices-based system that combines static code analysis, AI-powered language models, and multilingual NLP to provide intelligent debugging assistance, automatic documentation generation, and educational support for developers across India. The system architecture prioritizes scalability, language flexibility, and real-time responsiveness.

## Architecture

### High-Level Architecture

The system follows a microservices architecture with the following main components:

```
┌─────────────────────────────────────────────────────────────────┐
│                         API Gateway                              │
│                    (Authentication & Routing)                    │
└────────────┬────────────────────────────────────────────────────┘
             │
    ┌────────┴────────┬──────────────┬──────────────┬─────────────┐
    │                 │              │              │             │
┌───▼────┐    ┌──────▼─────┐  ┌────▼─────┐  ┌────▼─────┐  ┌───▼────┐
│ Code   │    │ AI Debug   │  │ Doc Gen  │  │ GitHub   │  │ Multi  │
│Analysis│    │ Service    │  │ Service  │  │ Service  │  │ lingual│
│Service │    │            │  │          │  │          │  │ Service│
└───┬────┘    └──────┬─────┘  └────┬─────┘  └────┬─────┘  └───┬────┘
    │                │              │              │             │
    └────────┬───────┴──────────────┴──────────────┴─────────────┘
             │
    ┌────────▼────────────────────────────────────────────┐
    │              Message Queue (RabbitMQ)               │
    └────────┬────────────────────────────────────────────┘
             │
    ┌────────▼────────┬──────────────┬──────────────┐
    │                 │              │              │
┌───▼────────┐  ┌────▼─────┐  ┌────▼──────┐  ┌───▼────────┐
│ PostgreSQL │  │  Redis   │  │ Vector DB │  │   S3/      │
│  (Metadata)│  │ (Cache)  │  │(Embeddings)│  │ MinIO      │
└────────────┘  └──────────┘  └───────────┘  └────────────┘
```

### Component Responsibilities

1. **API Gateway**: Request routing, authentication, rate limiting, and load balancing
2. **Code Analysis Service**: Static code analysis, error detection, and code metrics
3. **AI Debug Service**: AI-powered debugging, fix suggestions, and code explanations
4. **Doc Generation Service**: Automatic documentation generation from code
5. **GitHub Service**: Repository cloning, analysis, and integration
6. **Multilingual Service**: Translation and multilingual explanation generation
7. **Message Queue**: Asynchronous task processing for long-running operations
8. **PostgreSQL**: User data, session history, and analysis results
9. **Redis**: Session caching and real-time debugging state
10. **Vector DB**: Code embeddings for semantic search and similarity analysis
11. **S3/MinIO**: Storage for generated documentation and large codebases

## Components and Interfaces

### 1. API Gateway

**Technology**: Kong or AWS API Gateway

**Responsibilities**:
- Route requests to appropriate microservices
- Handle authentication using JWT tokens
- Implement rate limiting (100 requests/minute per user)
- Provide API versioning support

**Endpoints**:
```
POST   /api/v1/auth/login
POST   /api/v1/auth/register
POST   /api/v1/code/analyze
POST   /api/v1/code/debug
POST   /api/v1/code/explain
POST   /api/v1/docs/generate
POST   /api/v1/github/analyze
POST   /api/v1/session/create
GET    /api/v1/session/{id}
POST   /api/v1/session/{id}/message
PUT    /api/v1/user/preferences
```

### 2. Code Analysis Service

**Technology**: Python with Tree-sitter for parsing

**Core Modules**:

**Parser Module**:
```python
class CodeParser:
    def parse(code: str, language: str) -> AST
    def extract_functions(ast: AST) -> List[Function]
    def extract_classes(ast: AST) -> List[Class]
    def extract_imports(ast: AST) -> List[Import]
```

**Error Detector Module**:
```python
class ErrorDetector:
    def detect_syntax_errors(ast: AST) -> List[SyntaxError]
    def detect_logical_errors(ast: AST) -> List[LogicalError]
    def detect_security_vulnerabilities(ast: AST) -> List[SecurityIssue]
    def detect_runtime_errors(ast: AST) -> List[RuntimeError]
```

**Metrics Calculator Module**:
```python
class MetricsCalculator:
    def calculate_complexity(ast: AST) -> ComplexityMetrics
    def calculate_loc(code: str) -> int
    def detect_code_smells(ast: AST) -> List[CodeSmell]
    def calculate_duplication(files: List[File]) -> DuplicationReport
```

**Interfaces**:
```python
@dataclass
class AnalysisRequest:
    code: str
    language: str
    analysis_type: List[str]  # ['errors', 'metrics', 'security']
    user_id: str

@dataclass
class AnalysisResult:
    errors: List[Error]
    metrics: CodeMetrics
    security_issues: List[SecurityIssue]
    analysis_time: float
```

### 3. AI Debug Service

**Technology**: Python with LangChain and OpenAI/Anthropic APIs

**Core Modules**:

**LLM Manager**:
```python
class LLMManager:
    def initialize_model(model_name: str) -> LLM
    def generate_fix_suggestion(error: Error, context: CodeContext) -> FixSuggestion
    def explain_code(code: str, complexity_level: str) -> Explanation
    def generate_productivity_suggestions(code: str) -> List[Suggestion]
```

**Context Builder**:
```python
class ContextBuilder:
    def build_error_context(error: Error, code: str) -> ErrorContext
    def build_code_context(code: str, ast: AST) -> CodeContext
    def extract_relevant_snippets(code: str, line_number: int, radius: int) -> str
```

**Fix Generator**:
```python
class FixGenerator:
    def generate_fixes(error: Error, context: ErrorContext) -> List[FixSuggestion]
    def rank_fixes(fixes: List[FixSuggestion]) -> List[FixSuggestion]
    def apply_fix(code: str, fix: FixSuggestion) -> str
```

**Interfaces**:
```python
@dataclass
class DebugRequest:
    code: str
    errors: List[Error]
    language: str
    user_level: str  # 'beginner', 'intermediate', 'advanced'
    session_id: Optional[str]

@dataclass
class DebugResponse:
    fix_suggestions: List[FixSuggestion]
    explanations: List[Explanation]
    learning_resources: List[Resource]
```

### 4. Documentation Generation Service

**Technology**: Python with language-specific documentation parsers

**Core Modules**:

**Doc Extractor**:
```python
class DocExtractor:
    def extract_function_docs(function: Function) -> FunctionDoc
    def extract_class_docs(class_def: Class) -> ClassDoc
    def extract_module_docs(module: Module) -> ModuleDoc
    def extract_api_docs(routes: List[Route]) -> APIDoc
```

**Doc Generator**:
```python
class DocGenerator:
    def generate_markdown(docs: Documentation) -> str
    def generate_html(docs: Documentation) -> str
    def generate_pdf(docs: Documentation) -> bytes
    def generate_jsdoc(docs: Documentation) -> str
```

**Template Engine**:
```python
class TemplateEngine:
    def load_template(format: str, language: str) -> Template
    def render(template: Template, data: dict) -> str
```

**Interfaces**:
```python
@dataclass
class DocGenerationRequest:
    code: str
    language: str
    output_format: str  # 'markdown', 'html', 'pdf', 'jsdoc'
    include_examples: bool
    include_diagrams: bool

@dataclass
class Documentation:
    title: str
    overview: str
    functions: List[FunctionDoc]
    classes: List[ClassDoc]
    modules: List[ModuleDoc]
    generated_at: datetime
```

### 5. GitHub Service

**Technology**: Python with PyGithub library

**Core Modules**:

**Repository Manager**:
```python
class RepositoryManager:
    def clone_repository(url: str, auth_token: Optional[str]) -> Repository
    def analyze_structure(repo: Repository) -> RepoStructure
    def extract_readme(repo: Repository) -> str
    def analyze_commits(repo: Repository) -> CommitAnalysis
```

**Dependency Analyzer**:
```python
class DependencyAnalyzer:
    def detect_dependencies(repo: Repository) -> List[Dependency]
    def build_dependency_graph(deps: List[Dependency]) -> Graph
    def detect_vulnerabilities(deps: List[Dependency]) -> List[Vulnerability]
```

**Interfaces**:
```python
@dataclass
class GitHubAnalysisRequest:
    repository_url: str
    auth_token: Optional[str]
    analysis_depth: str  # 'quick', 'standard', 'deep'
    include_history: bool

@dataclass
class RepositoryAnalysis:
    summary: str
    languages: Dict[str, float]  # language -> percentage
    structure: RepoStructure
    quality_issues: List[Issue]
    contributor_guide: str
```

### 6. Multilingual Service

**Technology**: Python with IndicTrans2 for Indian languages

**Core Modules**:

**Translation Engine**:
```python
class TranslationEngine:
    def translate(text: str, source_lang: str, target_lang: str) -> str
    def translate_technical_term(term: str, target_lang: str) -> TranslatedTerm
    def preserve_code_snippets(text: str) -> Tuple[str, List[CodeSnippet]]
```

**Language Detector**:
```python
class LanguageDetector:
    def detect_language(text: str) -> str
    def get_user_preference(user_id: str) -> str
    def set_user_preference(user_id: str, language: str) -> None
```

**Terminology Manager**:
```python
class TerminologyManager:
    def load_technical_glossary(language: str) -> Dict[str, str]
    def add_term(english_term: str, translations: Dict[str, str]) -> None
    def get_translation(term: str, target_lang: str) -> Optional[str]
```

**Interfaces**:
```python
@dataclass
class TranslationRequest:
    text: str
    target_language: str
    preserve_technical_terms: bool
    context: Optional[str]

@dataclass
class TranslatedContent:
    translated_text: str
    technical_terms: List[TechnicalTerm]
    confidence_score: float
```

### 7. Session Manager

**Technology**: Python with Redis for state management

**Core Modules**:

**Session Handler**:
```python
class SessionHandler:
    def create_session(user_id: str) -> Session
    def get_session(session_id: str) -> Optional[Session]
    def update_session(session_id: str, data: dict) -> None
    def close_session(session_id: str) -> None
```

**Context Manager**:
```python
class ContextManager:
    def add_message(session_id: str, message: Message) -> None
    def get_conversation_history(session_id: str) -> List[Message]
    def get_relevant_context(session_id: str, query: str) -> Context
```

**Interfaces**:
```python
@dataclass
class Session:
    session_id: str
    user_id: str
    created_at: datetime
    last_active: datetime
    context: Dict[str, Any]
    conversation_history: List[Message]

@dataclass
class Message:
    role: str  # 'user' or 'assistant'
    content: str
    timestamp: datetime
    metadata: Dict[str, Any]
```

## Data Models

### Core Entities

**User**:
```python
@dataclass
class User:
    user_id: str
    email: str
    name: str
    language_preference: str
    skill_level: str  # 'beginner', 'intermediate', 'advanced'
    created_at: datetime
    preferences: UserPreferences
```

**Error**:
```python
@dataclass
class Error:
    error_id: str
    error_type: str  # 'syntax', 'logical', 'runtime', 'security'
    severity: str  # 'critical', 'high', 'medium', 'low'
    line_number: int
    column_number: int
    message: str
    code_snippet: str
```

**FixSuggestion**:
```python
@dataclass
class FixSuggestion:
    suggestion_id: str
    error_id: str
    description: str
    code_diff: str
    confidence_score: float
    explanation: str
    learning_resources: List[str]
```

**CodeMetrics**:
```python
@dataclass
class CodeMetrics:
    lines_of_code: int
    cyclomatic_complexity: int
    maintainability_index: float
    code_duplication_percentage: float
    test_coverage: Optional[float]
    dependencies_count: int
```

### Database Schema

**PostgreSQL Tables**:

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255),
    language_preference VARCHAR(50) DEFAULT 'en',
    skill_level VARCHAR(50) DEFAULT 'beginner',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE analysis_history (
    analysis_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    code_hash VARCHAR(64),
    language VARCHAR(50),
    analysis_type VARCHAR(50),
    result JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE sessions (
    session_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_active TIMESTAMP,
    status VARCHAR(50),
    context JSONB
);

CREATE TABLE error_patterns (
    pattern_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    error_type VARCHAR(100),
    frequency INT DEFAULT 1,
    last_occurred TIMESTAMP,
    resolved BOOLEAN DEFAULT FALSE
);
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Error Detection Completeness

*For any* valid Code_Snippet in a supported language, when analyzed by the Error Detector, all syntax errors present in the code should be included in the returned Error_Report.

**Validates: Requirements 2.1**

### Property 2: Fix Suggestion Validity

*For any* Error_Report generated by the system, each Fix_Suggestion should produce syntactically valid code when applied to the original Code_Snippet.

**Validates: Requirements 1.3**

### Property 3: Language Preference Consistency

*For any* User with a set Language_Preference, all generated Explanations and Error_Reports within a single session should be in the same target language.

**Validates: Requirements 6.2, 6.5**

### Property 4: Translation Preservation

*For any* technical term in an Explanation, when translated to a target language, translating back to English should preserve the original meaning or return the original English term.

**Validates: Requirements 6.3, 6.4**

### Property 5: Documentation Completeness

*For any* Code_Snippet containing functions with docstrings or comments, the generated Documentation should include entries for all public functions and classes.

**Validates: Requirements 4.2, 4.3**

### Property 6: Session Context Persistence

*For any* Debug_Session, when a User asks a follow-up question referencing a previous Error_Report, the System should have access to the complete conversation history from that session.

**Validates: Requirements 9.2, 9.5**

### Property 7: Error Prioritization Ordering

*For any* Error_Report containing multiple errors, the errors should be ordered such that all critical severity errors appear before high severity, high before medium, and medium before low.

**Validates: Requirements 1.4**

### Property 8: Code Metrics Invariant

*For any* Code_Snippet, calculating code metrics multiple times without modifying the code should produce identical CodeMetrics results.

**Validates: Requirements 5.4**

### Property 9: Repository Analysis Timeout

*For any* GitHub Repository with up to 10,000 lines of code, the analysis should complete within 60 seconds or return a partial result with a timeout indicator.

**Validates: Requirements 8.1**

### Property 10: Security Vulnerability Detection

*For any* Code_Snippet containing a known OWASP Top 10 vulnerability pattern, the System should detect and report it with appropriate severity rating.

**Validates: Requirements 10.1, 10.5**

### Property 11: Explanation Complexity Adaptation

*For any* Code_Snippet, when generating Explanations for different skill levels (beginner vs advanced), the beginner explanation should contain more detailed step-by-step breakdowns and simpler vocabulary.

**Validates: Requirements 3.5**

### Property 12: API Response Time

*For any* Code_Snippet under 1000 lines, the analysis and error detection should complete within 5 seconds.

**Validates: Requirements 1.1**

### Property 13: Export Format Validity

*For any* generated Documentation, when exported to a specified format (Markdown, HTML, PDF), the output should be valid according to that format's specification.

**Validates: Requirements 12.3**

### Property 14: Multilingual Term Consistency

*For any* technical term that appears multiple times in an Explanation, all occurrences should use the same translation in the target language.

**Validates: Requirements 6.5**

### Property 15: GitHub Authentication Handling

*For any* private GitHub Repository, when provided with valid authentication credentials, the System should successfully clone and analyze the repository; when credentials are invalid, it should return an authentication error without attempting analysis.

**Validates: Requirements 8.7**

## Error Handling

### Error Categories

1. **User Input Errors**:
   - Invalid code syntax (return detailed syntax error)
   - Unsupported programming language (return list of supported languages)
   - Malformed API requests (return 400 with validation errors)

2. **System Errors**:
   - LLM API failures (retry with exponential backoff, fallback to cached responses)
   - Database connection failures (return 503, queue request for retry)
   - Service timeout (return partial results with timeout indicator)

3. **External Service Errors**:
   - GitHub API rate limiting (return 429 with retry-after header)
   - Translation service failures (fallback to English)
   - Authentication failures (return 401 with clear error message)

### Error Response Format

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {
      "field": "specific_field",
      "reason": "detailed_reason"
    },
    "timestamp": "2024-01-15T10:30:00Z",
    "request_id": "uuid"
  }
}
```

### Retry Strategy

- **Transient Errors**: Retry up to 3 times with exponential backoff (1s, 2s, 4s)
- **Rate Limiting**: Respect retry-after headers, queue requests
- **Timeout Errors**: Return partial results if available, otherwise fail fast

### Fallback Mechanisms

1. **LLM Failures**: Use rule-based error detection and cached fix patterns
2. **Translation Failures**: Return content in English with apology message
3. **GitHub Failures**: Analyze local code only, skip repository features
4. **Database Failures**: Use in-memory cache for session data

## Testing Strategy

### Unit Testing

**Focus Areas**:
- Code parser correctness for each supported language
- Error detection accuracy for known error patterns
- Translation accuracy for technical terms
- Documentation generation for standard code structures
- API endpoint request/response validation

**Tools**: pytest, Jest, JUnit (language-specific)

**Coverage Target**: Minimum 80% code coverage

### Property-Based Testing

**Configuration**: Minimum 100 iterations per property test using Hypothesis (Python), fast-check (TypeScript)

**Property Tests**:

1. **Error Detection Completeness** (Property 1)
   - Generate random code with injected syntax errors
   - Verify all injected errors are detected
   - **Feature: ai-bharat-doc-debug-assistant, Property 1: Error Detection Completeness**

2. **Fix Suggestion Validity** (Property 2)
   - Generate random errors and fix suggestions
   - Apply fixes and verify resulting code parses successfully
   - **Feature: ai-bharat-doc-debug-assistant, Property 2: Fix Suggestion Validity**

3. **Language Preference Consistency** (Property 3)
   - Generate random session with multiple requests
   - Verify all responses use the same target language
   - **Feature: ai-bharat-doc-debug-assistant, Property 3: Language Preference Consistency**

4. **Translation Preservation** (Property 4)
   - Generate random technical terms
   - Translate to target language and back to English
   - Verify meaning preservation
   - **Feature: ai-bharat-doc-debug-assistant, Property 4: Translation Preservation**

5. **Documentation Completeness** (Property 5)
   - Generate random code with functions and classes
   - Verify all public entities appear in documentation
   - **Feature: ai-bharat-doc-debug-assistant, Property 5: Documentation Completeness**

6. **Session Context Persistence** (Property 6)
   - Generate random debug sessions with multiple messages
   - Verify conversation history is maintained
   - **Feature: ai-bharat-doc-debug-assistant, Property 6: Session Context Persistence**

7. **Error Prioritization Ordering** (Property 7)
   - Generate random error lists with mixed severities
   - Verify errors are sorted by severity correctly
   - **Feature: ai-bharat-doc-debug-assistant, Property 7: Error Prioritization Ordering**

8. **Code Metrics Invariant** (Property 8)
   - Generate random code snippets
   - Calculate metrics multiple times
   - Verify results are identical
   - **Feature: ai-bharat-doc-debug-assistant, Property 8: Code Metrics Invariant**

9. **Security Vulnerability Detection** (Property 10)
   - Generate code with known vulnerability patterns
   - Verify vulnerabilities are detected with correct severity
   - **Feature: ai-bharat-doc-debug-assistant, Property 10: Security Vulnerability Detection**

10. **Export Format Validity** (Property 13)
    - Generate random documentation
    - Export to each format
    - Verify format validity using validators
    - **Feature: ai-bharat-doc-debug-assistant, Property 13: Export Format Validity**

### Integration Testing

**Focus Areas**:
- End-to-end API workflows (submit code → receive analysis → apply fix)
- Service-to-service communication
- Database transactions and rollbacks
- Message queue processing
- Authentication and authorization flows

**Tools**: Postman/Newman, pytest with fixtures, Testcontainers

### Performance Testing

**Metrics**:
- API response time (p50, p95, p99)
- Throughput (requests per second)
- Error rate under load
- Resource utilization (CPU, memory, network)

**Tools**: Apache JMeter, Locust, k6

**Targets**:
- Code analysis: < 5 seconds for 1000 lines
- Repository analysis: < 60 seconds for 10,000 lines
- API throughput: > 100 requests/second
- Error rate: < 1% under normal load

### Security Testing

**Focus Areas**:
- Input validation and sanitization
- SQL injection prevention
- XSS prevention in generated documentation
- Authentication bypass attempts
- Rate limiting effectiveness

**Tools**: OWASP ZAP, Burp Suite, Bandit (Python security linter)

### Multilingual Testing

**Focus Areas**:
- Translation accuracy for technical terms
- Character encoding handling (Unicode, Devanagari, Tamil script)
- Right-to-left language support (if applicable)
- Language detection accuracy

**Test Data**: Curated dataset of technical explanations in all supported languages

### Accessibility Testing

**Focus Areas**:
- Generated HTML documentation meets WCAG 2.1 AA standards
- API responses include proper content-type headers
- Error messages are screen-reader friendly

**Tools**: axe-core, WAVE, Lighthouse

