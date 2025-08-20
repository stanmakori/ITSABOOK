# Nyabitono Bank Lite Payment API

A comprehensive guide to implementing the Nyabitono Bank Lite Payment API using OpenAPI 3.0 specification with code generation examples.

## Why API Design-First Matters: A Critical Comparison

### The Traditional "Code-First" Problem

Many developers think: *"Why not just create Java APIs and share the JAR files or endpoints?"* Here's why this approach fails in real-world scenarios:

#### **Scenario: Traditional Java-First Approach**

```java
// Backend team creates this Java API
@RestController
public class PaymentController {
    
    @PostMapping("/sendMoney")  // Inconsistent naming
    public String processPayment(
        @RequestParam String phone,    // Loose validation
        @RequestParam double amount,   // No currency specified
        @RequestParam String desc      // Abbreviated parameter
    ) {
        // Business logic here
        return "Payment processed: " + generateId();  // Inconsistent response format
    }
    
    @GetMapping("/checkStatus/{id}")
    public Map<String, Object> getStatus(@PathVariable String id) {
        Map<String, Object> result = new HashMap<>();
        result.put("status", "SUCCESS");  // Inconsistent with other responses
        result.put("txnId", id);          // Different field naming
        return result;
    }
}
```

#### **Problems This Creates:**

### **1. Integration Chaos**

**Python Team Struggles:**
```python
# Python team has documentation, but it's often incomplete or inconsistent
import requests

# Documentation says: "POST /sendMoney with phone, amount, desc"
# But doesn't specify:
# - Exact parameter names (phone vs phoneNumber vs recipient_phone?)
# - Data types (string vs integer for phone?)
# - Validation rules (phone format? amount limits?)
# - Request format (JSON body vs form data vs query params?)
# - Response structure (JSON object vs plain string?)

response = requests.post("http://java-service/sendMoney", data={
    "phone": "254712345678",    # Guessing parameter name from docs
    "amount": 150.50,           # Docs don't specify decimal precision
    "desc": "Payment"           # Docs say "desc" but is it required?
})

# Documentation doesn't specify response format
if response.status_code == 200:
    result = response.text  # Is this JSON? Plain text? How to parse?
    # How do I extract the transaction ID from the response?
```

**JavaScript Team Struggles:**
```javascript
// JavaScript team has the same documentation but interprets it differently
fetch('/sendMoney', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
        phoneNumber: "254712345678",  // They assume camelCase naming
        paymentAmount: 150.50,        // They expand "amount" to be clearer
        description: "Payment"        // They expand "desc" to full word
    })
})
.then(response => {
    // Documentation doesn't specify error handling
    if (response.ok) {
        return response.json();  // Assuming JSON, but docs unclear
    }
    throw new Error('Payment failed');  // Generic error handling
});
```

### **2. Documentation Ambiguity**

**What Developers Share:**
```markdown
# Payment API Documentation (manually written)
## Send Money
- URL: POST /sendMoney
- Parameters: phone (string), amount (number), desc (string, optional)
- Returns: Success message with transaction ID

## Check Status  
- URL: GET /checkStatus/{id}
- Parameters: id (string) - transaction identifier
- Returns: Status object with transaction details

# Problems with this documentation:
# ❌ Ambiguous data formats (JSON body? Form data? Query params?)
# ❌ Missing validation rules (phone format? amount limits?)
# ❌ Unclear response structure (what does "status object" contain?)
# ❌ No error scenarios (what if payment fails?)
# ❌ No examples (developers interpret differently)
```

**Real Documentation Challenges:**
```java
// Code evolves but documentation lags behind
@PostMapping("/sendMoney")
public ResponseEntity<PaymentResponse> processPayment(
    @RequestBody PaymentRequest request  // Changed from @RequestParam!
) {
    // Documentation still shows old parameter format
    // New validation rules not documented
    // Response format changed but docs not updated
}
```

### **3. Version Management Hell**

```java
// Version 1.0
@PostMapping("/sendMoney")
public String processPayment(@RequestParam String phone, @RequestParam double amount) {
    return "Payment processed: " + id;
}

// Version 1.1 - Breaking change!
@PostMapping("/sendMoney") 
public PaymentResponse processPayment(@RequestBody PaymentRequest request) {
    return new PaymentResponse(id, "SUCCESS", timestamp);
}

// Now all existing clients break!
// No clear migration path
// No backward compatibility strategy
```

---

## The Design-First OpenAPI Solution

### **Single Source of Truth**

```yaml
# ONE specification that everyone follows
paths:
  /payments:
    post:
      summary: Initiate a new mobile money payment
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/PaymentRequest'
            examples:
              basic_payment:
                value:
                  amount: 150.50
                  recipientPhoneNumber: "254712345678"
                  currency: "KES"
                  remarks: "Payment for groceries"
      responses:
        '202':
          description: Payment request accepted
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PaymentSuccessResponse'
              examples:
                success_response:
                  value:
                    transactionId: "TXN_a1b2c3d4e5"
                    status: "Submitted"
                    message: "Payment request received successfully"
```

### **Consistent Multi-Language Implementation**

**Generated Java (Backend):**
```java
// Auto-generated from OAS - perfect consistency
@RestController
public class PaymentsController implements PaymentsApi {
    
    @Override
    public ResponseEntity<PaymentSuccessResponse> initiatePayment(
        @Valid @RequestBody PaymentRequest paymentRequest
    ) {
        // Only business logic needed - structure is guaranteed
        PaymentSuccessResponse response = paymentService.processPayment(paymentRequest);
        return ResponseEntity.accepted().body(response);
    }
}

// Generated models with validation
public class PaymentRequest {
    @NotNull
    @DecimalMin("1.0")
    private BigDecimal amount;
    
    @NotNull
    @Pattern(regexp = "^254[0-9]{9}$")
    private String recipientPhoneNumber;
    
    @NotNull
    private String currency = "KES";
    
    // Perfect consistency guaranteed
}
```

**Generated Python Client:**
```python
# Auto-generated from same OAS - zero guesswork
from nyabitono_client import NyabitonoClient
from nyabitono_client.models import PaymentRequest

client = NyabitonoClient(host="https://api.nyabitonobank.com/v1")

# Type-safe, validated request
payment_request = PaymentRequest(
    amount=150.50,                          # Validated: must be > 0
    recipient_phone_number="254712345678",  # Validated: must match pattern
    currency="KES",                         # Default value from OAS
    remarks="Payment for groceries"
)

try:
    response = client.payments.initiate_payment(payment_request)
    print(f"Transaction ID: {response.transaction_id}")  # Known response structure
except ValidationException as e:
    print(f"Invalid request: {e}")  # Proper error handling
except ApiException as e:
    print(f"API error: {e}")
```

**Generated JavaScript Client:**
```javascript
// Auto-generated from same OAS - perfect consistency
import { NyabitonoApi, PaymentRequest } from 'nyabitono-client';

const api = new NyabitonoApi();

const paymentRequest = new PaymentRequest({
    amount: 150.50,                          // Same validation as other languages
    recipientPhoneNumber: "254712345678",    // Same field names
    currency: "KES",                         // Same defaults
    remarks: "Payment for groceries"
});

try {
    const response = await api.initiatePayment(paymentRequest);
    console.log(`Transaction ID: ${response.transactionId}`);  // Consistent naming
} catch (error) {
    console.error('Payment failed:', error);  // Proper error handling
}
```

### **Perfect Documentation**

```yaml
# Documentation is automatically generated and always accurate
# Interactive examples that actually work
# No manual maintenance required
# Swagger UI shows:
# - All endpoints with examples
# - Request/response schemas
# - Validation rules
# - Error codes
# - Authentication requirements
```

### **Seamless Version Management**

```yaml
# Version 1.0.0
openapi: 3.0.0
info:
  version: 1.0.0

# Version 1.1.0 - Backward compatible
openapi: 3.0.0  
info:
  version: 1.1.0
# New optional fields added
# Existing fields unchanged
# All existing clients continue working

# Version 2.0.0 - Breaking changes
openapi: 3.0.0
info:
  version: 2.0.0
# Clear migration guide
# Side-by-side deployment possible
# Deprecation timeline clearly defined
```

## **Real-World Impact Comparison**

| Aspect | Code-First (Java API) | Design-First (OpenAPI) |
|--------|----------------------|-------------------------|
| **Integration Time** | 2-4 weeks per client | 2-4 hours per client |
| **Documentation** | Manual, often outdated | Auto-generated, always current |
| **Multi-language Support** | Manual recreation, inconsistent | Auto-generated, identical behavior |
| **Error Prone** | High (mismatched assumptions) | Low (contract enforced) |
| **Onboarding New Developers** | Days (learning custom API) | Minutes (standard patterns) |
| **API Evolution** | Breaking changes common | Backward compatibility enforced |
| **Testing** | Manual, incomplete coverage | Generated test cases |
| **Client SDK Quality** | Varies by team skill | Professional grade, consistent |

## **Business Impact**

### **Without Design-First:**
- **Developer Support Overhead:** Constant integration issues and questions
- **Slow Partner Onboarding:** Weeks to integrate new clients
- **Version Management Chaos:** Breaking changes affect all partners
- **Poor Developer Experience:** Partners struggle with integration

### **With Design-First OpenAPI:**
- **Self-Service Integration:** Partners integrate independently
- **Rapid Ecosystem Growth:** Easy onboarding attracts more partners
- **Reliable API Evolution:** Smooth version transitions
- **Professional Developer Experience:** Partners love working with your APIs

## **The Netflix/Stripe Success Pattern**

Companies like **Netflix**, **Stripe**, and **Amazon Web Services** use OpenAPI design-first because:

```yaml
# They serve thousands of developers across hundreds of companies
# Consistency and reliability are critical
# Self-service integration reduces support costs
# Professional SDKs increase adoption
# Clear contracts enable massive scale
```

**Bottom Line:** Design-first with OpenAPI transforms your API from a **internal Java service** into a **professional platform** that scales across languages, teams, and companies.

## Overview

The Nyabitono Bank Lite Payment API enables authorized third-party applications to:

- Initiate mobile money payments
- Check transaction status
- Retrieve account balance on behalf of Nyabitono Bank Lite customers

All payments are processed securely via banking partners with OAuth2 authentication.

## API Specification

The complete OpenAPI 3.0 specification for the Nyabitono Bank Lite Payment API:

```yaml
openapi: 3.0.0
info:
  title: Nyabitono Bank Lite Payment API
  description: |
    # Nyabitono Bank Lite API
    This API allows authorized third-party applications to initiate mobile money payments,
    check transaction status, and retrieve account balance on behalf of Nyabitono Bank Lite customers.
    All payments are processed securely via our banking partners.
  version: 1.0.0
  contact:
    name: Nyabitono Bank Lite API Team
    url: https://developer.nyabitonobank.com/support
    email: api-support@nyabitonobank.com
  license:
    name: Proprietary
    url: https://developer.nyabitonobank.com/terms

servers:
  - url: https://api-sandbox.nyabitonobank.com/v1
    description: Sandbox testing environment (use for development)
  - url: https://api.nyabitonobank.com/v1
    description: Production environment for live transactions.

components:
  schemas:
    PaymentRequest:
      type: object
      required:
        - amount
        - recipientPhoneNumber
      properties:
        amount:
          type: number
          format: float
          minimum: 1
          example: 150.50
          description: The amount to be paid. Must be greater than 0.
        recipientPhoneNumber:
          type: string
          pattern: '^254[0-9]{9}$'
          example: "254712345678"
          description: Recipient's phone number. Must be a valid Kenyan format (254...).
        currency:
          type: string
          example: KES
          default: KES
          description: Currency code. Only KES is supported.
        paymentReference:
          type: string
          example: "INV-2023-789"
          description: A client-generated reference for tracking this payment.
        remarks:
          type: string
          example: "Payment for groceries"
          description: A description of the payment.

    PaymentSuccessResponse:
      type: object
      properties:
        transactionId:
          type: string
          example: "TXN_a1b2c3d4e5"
          description: A unique identifier for the successful transaction.
        status:
          type: string
          example: "Submitted"
          description: The current status of the payment request.
        message:
          type: string
          example: "Payment request received successfully."
        requestReference:
          type: string
          example: "req_987654321"
          description: A unique identifier for this API request for support purposes.

    Transaction:
      type: object
      properties:
        id:
          type: string
          example: "TXN_a1b2c3d4e5"
        amount:
          type: number
          example: 150.50
        type:
          type: string
          example: "DEBIT"
          enum: [DEBIT, CREDIT]
        recipient:
          type: string
          example: "254712345678"
        status:
          type: string
          example: "SUCCESS"
          enum: [SUBMITTED, PROCESSING, SUCCESS, FAILED, REVERSED]
        timestamp:
          type: string
          format: date-time
          example: "2023-10-27T10:30:00Z"

    APIError:
      type: object
      properties:
        errorCode:
          type: string
          example: "VALIDATION_ERROR"
        errorMessage:
          type: string
          example: "The field 'recipientPhoneNumber' is invalid."
        requestId:
          type: string
          example: "req_abcdef123456"
          description: A unique identifier for the request. Provide this when contacting support.

  securitySchemes:
    OAuth2:
      type: oauth2
      flows:
        clientCredentials:
          tokenUrl: https://api.nyabitonobank.com/oauth/token
          scopes:
            payments:send: Initiate a payment
            transactions:read: View transaction history
            balance:read: Check account balance

  responses:
    BadRequestError:
      description: Invalid request parameters or payload.
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/APIError'
    UnauthorizedError:
      description: Access token is missing, invalid, or expired.
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/APIError'
    NotFoundError:
      description: The requested resource was not found.
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/APIError'

  parameters:
    TransactionIdPathParam:
      name: transactionId
      in: path
      required: true
      schema:
        type: string
      description: The unique identifier for the transaction.

paths:
  /payments:
    post:
      tags:
        - Payments
      summary: Initiate a new mobile money payment
      description: Submit a request to send money to a recipient's mobile phone number.
      operationId: initiatePayment
      security:
        - OAuth2: 
            - payments:send
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/PaymentRequest'
      responses:
        '202':
          description: Payment request accepted for processing.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PaymentSuccessResponse'
        '400':
          $ref: '#/components/responses/BadRequestError'
        '401':
          $ref: '#/components/responses/UnauthorizedError'
        '500':
          description: Internal server error.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/APIError'

  /transactions/{transactionId}:
    get:
      tags:
        - Transactions
      summary: Get transaction status by ID
      description: Retrieve the current status and details of a specific transaction.
      operationId: getTransaction
      security:
        - OAuth2: 
            - transactions:read
      parameters:
        - $ref: '#/components/parameters/TransactionIdPathParam'
      responses:
        '200':
          description: Successful operation. Returns transaction details.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Transaction'
        '401':
          $ref: '#/components/responses/UnauthorizedError'
        '404':
          $ref: '#/components/responses/NotFoundError'

tags:
  - name: Payments
    description: Everything about initiating and managing payments.
  - name: Transactions
    description: Operations for retrieving transaction history and status.
```

## Code Generation Setup

### Prerequisites

Before generating server and client code, ensure you have:

- **Java JDK 17+** with `JAVA_HOME` set correctly
- **Node.js and npm** for installing OpenAPI Generator
- **Maven** for building generated Java server code

### Installation

Install the OpenAPI Generator CLI globally:

```bash
npm install @openapitools/openapi-generator-cli -g
```

### Setup Steps

1. Save the OpenAPI specification above to a file named `nyabitono-oas.yaml`
2. Verify the installation:
   ```bash
   openapi-generator-cli version
   ```

## Server Implementation

### Option A: Using OpenAPI Generator CLI

Generate a Spring Boot server with the following command:

```bash
openapi-generator-cli generate \
  -i nyabitono-oas.yaml \
  -g spring \
  -o ./nyabitono-server \
  --additional-properties=packageName=com.nyabitonobank.api,interfaceOnly=true,useSpringBoot3=true
```

**Command flags explained:**
- `-i`: Input specification file
- `-g`: Generator name (spring for Spring Boot)
- `-o`: Output directory for generated code
- `--additional-properties`: Key configuration options

Build the generated server:

```bash
cd nyabitono-server
mvn clean package
```

### Option B: Using Maven Plugin (Recommended)

Add this plugin to your `pom.xml`:

```xml
<plugin>
    <groupId>org.openapitools</groupId>
    <artifactId>openapi-generator-maven-plugin</artifactId>
    <version>6.6.0</version>
    <executions>
        <execution>
            <goals>
                <goal>generate</goal>
            </goals>
            <configuration>
                <inputSpec>${project.basedir}/src/main/resources/nyabitono-oas.yaml</inputSpec>
                <generatorName>spring</generatorName>
                <configOptions>
                    <packageName>com.nyabitonobank.api</packageName>
                    <interfaceOnly>true</interfaceOnly>
                    <useSpringBoot3>true</useSpringBoot3>
                    <useBeanValidation>true</useBeanValidation>
                    <dateLibrary>java8</dateLibrary>
                </configOptions>
            </configuration>
        </execution>
    </executions>
</plugin>
```

Generate the code:

```bash
mvn clean generate-sources
```

### Generated Components

The generator creates:
- **API Interfaces**: Spring `@Controller` interfaces (e.g., `PaymentsApi.java`)
- **Model Classes**: Java POJOs for all schemas (e.g., `PaymentRequest.java`)
- **Configuration**: Spring configuration and OpenAPI documentation support

### Implementation Example

Create a controller class to implement the generated interface:

```java
import com.nyabitonobank.api.PaymentsApi;
import com.nyabitonobank.api.model.PaymentRequest;
import com.nyabitonobank.api.model.PaymentSuccessResponse;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class PaymentsController implements PaymentsApi {

    @Override
    public ResponseEntity<PaymentSuccessResponse> initiatePayment(PaymentRequest paymentRequest) {
        // 1. Implement your business logic here
        // 2. Validate, process, and persist the payment
        // 3. Return the response
        
        PaymentSuccessResponse response = new PaymentSuccessResponse();
        response.setTransactionId("TXN_12345");
        response.setStatus("Submitted");
        response.setMessage("Payment request received successfully.");
        response.setRequestReference("req_987654321");
        
        return ResponseEntity.accepted().body(response);
    }
}
```

## Client SDK Generation

Generate client SDKs for various programming languages to consume the API.

### Python Client

Generate the Python client:

```bash
openapi-generator-cli generate \
  -i nyabitono-oas.yaml \
  -g python \
  -o ./nyabitono-python-client \
  --additional-properties=packageName=nyabitono_client
```

**Usage example:**

```python
import nyabitono_client
from nyabitono_client.rest import ApiException
from nyabitono_client.models import PaymentRequest

# Configure the client
configuration = nyabitono_client.Configuration(host="https://api-sandbox.nyabitonobank.com/v1")
api_client = nyabitono_client.ApiClient(configuration)
api_instance = nyabitono_client.PaymentsApi(api_client)

# Create a payment request
payment_request = PaymentRequest(
    amount=150.50,
    recipient_phone_number="254712345678",
    remarks="Payment for groceries"
)

try:
    # Initiate payment
    api_response = api_instance.initiate_payment(payment_request)
    print(f"Payment initiated: {api_response}")
except ApiException as e:
    print(f"Exception: {e}")
```

### TypeScript (Angular) Client

Generate the TypeScript client:

```bash
openapi-generator-cli generate \
  -i nyabitono-oas.yaml \
  -g typescript-angular \
  -o ./nyabitono-angular-client \
  --additional-properties=ngVersion=15.0.0,npmName=nyabitono-client
```

### Other Supported Clients

- **JavaScript**: `-g javascript`
- **Java**: `-g java` (Retrofit2, OkHttp, etc.)
- **Go**: `-g go`
- **C#**: `-g csharp`
- **PHP**: `-g php`
- **Ruby**: `-g ruby`

To see all available generators:

```bash
openapi-generator-cli list
```

## API Gateway Integration with WSO2

### Understanding the API Gateway Pattern

The OpenAPI specification serves **three different purposes** in a complete API ecosystem:

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   OAS File      │    │  WSO2 Gateway   │    │ Backend Service │
│ (Contract)      │───▶│  (Proxy Layer)  │───▶│ (Implementation)│
│                 │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
        │                       │                       │
        ▼                       ▼                       ▼
   Generates:             Creates:               Implements:
   • Client SDKs          • API Proxy            • Business Logic
   • Documentation        • Security             • Data Persistence
   • Server Templates     • Rate Limiting        • External APIs
                         • Monitoring           • Validation
```

### Key Understanding: URL Redirection

**❌ Common Misconception:**
```yaml
# Students often think clients connect directly to these URLs:
servers:
  - url: https://api.nyabitonobank.com/v1  # NOT where clients connect!
```

**✅ Reality with API Gateway:**
```bash
# 1. OAS defines the contract
servers:
  - url: https://api.nyabitonobank.com/v1  # Contract specification

# 2. Backend implements at internal URL
Backend runs at: http://internal-payment-service:8080

# 3. WSO2 creates public gateway URL
WSO2 Gateway URL: https://wso2-gateway.nyabitonobank.com/nyabitono/v1

# 4. Clients connect to WSO2, NOT the original OAS URLs
Client connects to: https://wso2-gateway.nyabitonobank.com/nyabitono/v1
```

### Complete Implementation Flow

#### **Phase 1: Development Team Tasks**

**Backend Team:**
```bash
# 1. Generate server code from OAS
openapi-generator-cli generate -g spring -i nyabitono-oas.yaml -o ./backend

# 2. Implement business logic
# 3. Deploy backend service at: http://internal-service:8080
# 4. Backend serves endpoints like: POST http://internal-service:8080/payments
```

**Client SDK Generation:**
```bash
# Generate client SDKs (but they'll point to WSO2 later)
openapi-generator-cli generate -g python -i nyabitono-oas.yaml
openapi-generator-cli generate -g javascript -i nyabitono-oas.yaml
```

#### **Phase 2: WSO2 Gateway Configuration**

**Import OAS to WSO2:**
```yaml
# WSO2 API Manager Configuration:
API Import: nyabitono-oas.yaml
Gateway Context: /nyabitono/v1
Backend Endpoint: http://internal-service:8080  # Maps to actual backend
Public URL: https://wso2-gateway.nyabitonobank.com/nyabitono/v1
```

**WSO2 Creates Runtime Proxy:**
```
Request Flow:
Client → WSO2 Gateway → Backend Service

1. POST https://wso2-gateway.nyabitonobank.com/nyabitono/v1/payments
   ↓
2. WSO2 validates against OAS schema
   ↓
3. WSO2 forwards to: http://internal-service:8080/payments
   ↓
4. Backend processes and responds
   ↓
5. WSO2 validates response against OAS
   ↓
6. WSO2 returns to client
```

#### **Phase 3: Client Configuration Update**

**Generated clients must be reconfigured:**
```python
# ❌ Generated client points to OAS servers (won't work):
client = NyabitonoClient(host="https://api.nyabitonobank.com/v1")

# ✅ Production client points to WSO2 Gateway:
client = NyabitonoClient(host="https://wso2-gateway.nyabitonobank.com/nyabitono/v1")
```

### Real-World Architecture

```
┌─────────────────┐
│ Third-Party App │
│                 │
└─────────┬───────┘
          │ HTTPS Request to WSO2
          ▼
┌─────────────────┐     ┌─────────────────┐
│  WSO2 Gateway   │────▶│ OAuth2 Server   │
│ Runtime Proxy   │     │ (Authentication)│
└─────────┬───────┘     └─────────────────┘
          │ Internal HTTP
          ▼
┌─────────────────┐     ┌─────────────────┐
│ Backend Service │────▶│    Database     │
│ (Generated +    │     │                 │
│  Custom Logic)  │     └─────────────────┘
└─────────────────┘
```

### WSO2 Developer Portal Experience

**What Third-Party Developers See:**
1. **API Documentation:** Generated from your OAS file
2. **Interactive Console:** Test APIs directly in browser
3. **SDK Downloads:** Auto-generated clients for multiple languages
4. **Subscription Management:** API keys and rate limits
5. **Usage Analytics:** Track their API consumption

**Developer Registration Flow:**
```bash
# 1. Developer registers at WSO2 Developer Portal
https://developer-portal.nyabitonobank.com

# 2. Creates application and gets credentials
Client ID: abc123
Client Secret: xyz789

# 3. Downloads SDK (pre-configured for WSO2 endpoint)
pip install nyabitono-api-client

# 4. Uses client pointing to WSO2 Gateway
from nyabitono_client import NyabitonoClient
client = NyabitonoClient(
    host="https://wso2-gateway.nyabitonobank.com/nyabitono/v1",
    client_id="abc123",
    client_secret="xyz789"
)
```

### Testing and Validation

#### **Direct Backend Testing** (Development Phase)
```bash
# Test backend service directly (before WSO2)
curl -X POST http://internal-service:8080/payments \
  -H "Content-Type: application/json" \
  -d '{"amount": 100, "recipientPhoneNumber": "254712345678"}'
```

#### **WSO2 Gateway Testing** (Production)
```bash
# Test through WSO2 Gateway (production flow)
curl -X POST https://wso2-gateway.nyabitonobank.com/nyabitono/v1/payments \
  -H "Authorization: Bearer WSO2_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"amount": 100, "recipientPhoneNumber": "254712345678"}'
```

#### **Interactive Documentation**
```bash
# WSO2 provides built-in API console
# No need for separate Swagger UI setup
Access at: https://developer-portal.nyabitonobank.com
```

### Key Benefits of This Architecture

| Aspect | Without API Gateway | With WSO2 Gateway |
|--------|-------------------|-------------------|
| **Client URLs** | Point directly to backend | Point to centralized gateway |
| **Security** | Implemented in each service | Centralized in gateway |
| **Rate Limiting** | Custom code in backend | Policy-based in WSO2 |
| **Monitoring** | Custom logging/metrics | Built-in analytics |
| **Documentation** | Manual maintenance | Auto-generated from OAS |
| **Client Onboarding** | Manual process | Self-service portal |
| **API Versioning** | Complex deployment | Gateway-managed routing |

## Examples

### Complete Payment Flow

1. **Authenticate** using OAuth2 client credentials
2. **Initiate Payment** via POST `/payments`
3. **Check Status** via GET `/transactions/{transactionId}`

### Error Handling

The API provides structured error responses:

```json
{
  "errorCode": "VALIDATION_ERROR",
  "errorMessage": "The field 'recipientPhoneNumber' is invalid.",
  "requestId": "req_abcdef123456"
}
```

## Support

For API support and questions:

- **Documentation**: [https://developer.nyabitonobank.com/support](https://developer.nyabitonobank.com/support)
- **Email**: api-support@nyabitonobank.com
- **Terms**: [https://developer.nyabitonobank.com/terms](https://developer.nyabitonobank.com/terms)

---

This documentation provides a complete, self-contained guide for the Nyabitono Bank Lite Payment API, including the full OpenAPI contract and practical instructions for generating both server-side and client-side code. The generated code handles all the boilerplate, allowing developers to focus on implementing unique business logic.
