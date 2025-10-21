
# REST Client API Documentation

## Overview
This document provides a comprehensive guide to the REST client implementation for the Quiet Lawn CRM application, detailing HTTP methods, parameters, and available API endpoints.

---

## HTTP Method Types

### GET
- **Purpose**: Retrieve data from the server
- **Characteristics**: 
  - Read-only operations
  - Parameters passed via URL (path/query parameters)
  - Idempotent (multiple identical requests produce the same result)
  - Cacheable

### POST
- **Purpose**: Create new resources or submit data
- **Characteristics**:
  - Creates new entities
  - Parameters passed in request body
  - Not idempotent
  - Can include complex data structures

### PATCH
- **Purpose**: Partially update existing resources
- **Characteristics**:
  - Modifies specific fields of existing data
  - Parameters passed in request body
  - Idempotent
  - Only sends changed fields

### DELETE
- **Purpose**: Remove resources from the server
- **Characteristics**:
  - Deletes existing entities
  - Parameters typically passed via path/query
  - Idempotent

---

## Parameter Types

### @Path
- **Usage**: Required path parameters embedded in the URL
- **Example**: `@Path('businessId') required String businessId`
- **Purpose**: Identifies specific resources in the URL structure

### @Query
- **Usage**: Optional/required query parameters appended to URL
- **Example**: `@Query('search') String? search`
- **Purpose**: Filtering, pagination, searching, and additional options

### @Body
- **Usage**: Request body data for POST/PATCH operations
- **Example**: `@Body() required Map<String, dynamic> body`
- **Purpose**: Send complex data structures

### @Part (Multipart)
- **Usage**: File upload and multipart form data
- **Example**: `@Part(name: 'files') required List<MultipartFile> files`
- **Purpose**: Upload files along with metadata

---

## Available API Methods

### Authentication

#### login
- **Method**: POST
- **Endpoint**: `/auth/login`
- **Parameters**: `LoginRequestModel` (body)
- **Purpose**: Authenticate user credentials

#### forgotPassword
- **Method**: POST
- **Endpoint**: `/auth/forgot-password`
- **Parameters**: `ForgotPasswordRequestModel` (body)
- **Purpose**: Initiate password recovery

#### logout
- **Method**: POST
- **Endpoint**: `/auth/logout`
- **Parameters**: None
- **Purpose**: End user session

#### switchBusiness
- **Method**: POST
- **Endpoint**: `/auth/switch-business`
- **Parameters**: `Map<String, dynamic>` (body)
- **Purpose**: Switch active business context

---

### Profile

#### getInfo
- **Method**: GET
- **Endpoint**: `/me`
- **Parameters**: None
- **Purpose**: Retrieve current user information

---

### Job Routes

#### getJobRoutes
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/job-routes`
- **Parameters**:
  - `businessId` (path, required)
  - `page` (query, optional)
  - `limit` (query, optional)
  - `date` (query, optional)
- **Purpose**: Fetch job routes with pagination and date filtering

---

### Jobs

#### getJobs
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/jobs/{id}`
- **Parameters**:
  - `businessId` (path, required)
  - `id` (path, required)
- **Purpose**: Retrieve specific job information

#### getJobDetails
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/jobs/{jobId}`
- **Parameters**:
  - `businessId` (path, required)
  - `jobId` (path, required)
- **Purpose**: Get detailed job information

#### updateJobDetails
- **Method**: PATCH
- **Endpoint**: `/businesses/{businessId}/jobs/{jobId}`
- **Parameters**:
  - `businessId` (path, required)
  - `jobId` (path, required)
  - `body` (body, required)
- **Purpose**: Update job details

#### uploadJobAttachments
- **Method**: POST (Multipart)
- **Endpoint**: `/businesses/{businessId}/jobs/attachments`
- **Parameters**:
  - `businessId` (path, required)
  - `files` (multipart, required)
  - `featureName` (multipart, required)
- **Purpose**: Upload files to job

#### deleteJobAttachment
- **Method**: DELETE
- **Endpoint**: `/businesses/{businessId}/jobs/attachments`
- **Parameters**:
  - `businessId` (path, required)
  - `fileKey` (query, required)
  - `featureName` (query, required)
- **Purpose**: Remove job attachment

---

### Customers

#### fetchCustomers
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/customers`
- **Parameters**:
  - `businessId` (path, required)
  - `search` (query, optional)
  - `page` (query, optional)
  - `limit` (query, optional)
- **Purpose**: List customers with search and pagination

#### createCustomer
- **Method**: POST
- **Endpoint**: `/businesses/{businessId}/customers`
- **Parameters**:
  - `businessId` (path, required)
  - `body` (body, required)
- **Purpose**: Create new customer

#### getCustomerDetails
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/customers/{customerId}`
- **Parameters**:
  - `businessId` (path, required)
  - `customerId` (path, required)
- **Purpose**: Get detailed customer information

#### fetchCustomerOrders
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/customers/{customerId}/orders`
- **Parameters**:
  - `businessId` (path, required)
  - `customerId` (path, required)
  - `page` (query, optional)
- **Purpose**: Retrieve customer order history

---

### Services

#### fetchServices
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/services`
- **Parameters**:
  - `businessId` (path, required)
  - `addressId` (query, required)
- **Purpose**: Get available services for address

#### fetchOneTimeServices
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/one-time-services`
- **Parameters**:
  - `businessId` (path, required)
  - `search` (query, optional)
  - `page` (query, optional)
  - `limit` (query, optional)
- **Purpose**: List one-time service offerings

---

### Address

#### getPropertyInfo
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/property-info`
- **Parameters**:
  - `businessId` (path, required)
  - `address` (query, required)
- **Purpose**: Retrieve property information by address

---

### Comments

#### getJobComments
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/jobs/{jobId}/comments`
- **Parameters**:
  - `businessId` (path, required)
  - `jobId` (path, required)
- **Purpose**: Fetch job comments

#### createJobComment
- **Method**: POST
- **Endpoint**: `/businesses/{businessId}/jobs/{jobId}/comments`
- **Parameters**:
  - `businessId` (path, required)
  - `jobId` (path, required)
  - `body` (body, required)
- **Purpose**: Add comment to job

#### getLeadComments
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/leads/{leadId}/comments`
- **Parameters**:
  - `businessId` (path, required)
  - `leadId` (path, required)
- **Purpose**: Fetch lead comments

#### createLeadComment
- **Method**: POST
- **Endpoint**: `/businesses/{businessId}/leads/{leadId}/comments`
- **Parameters**:
  - `businessId` (path, required)
  - `leadId` (path, required)
  - `body` (body, required)
- **Purpose**: Add comment to lead

#### getVisitComments
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/visits/{visitId}/comments`
- **Parameters**:
  - `businessId` (path, required)
  - `visitId` (path, required)
- **Purpose**: Fetch visit comments

#### createVisitComment
- **Method**: POST
- **Endpoint**: `/businesses/{businessId}/visits/{visitId}/comments`
- **Parameters**:
  - `businessId` (path, required)
  - `visitId` (path, required)
  - `body` (body, required)
- **Purpose**: Add comment to visit

#### getTaskComments
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/tasks/{taskId}/comments`
- **Parameters**:
  - `businessId` (path, required)
  - `taskId` (path, required)
- **Purpose**: Fetch task comments

#### createTaskComment
- **Method**: POST
- **Endpoint**: `/businesses/{businessId}/tasks/{taskId}/comments`
- **Parameters**:
  - `businessId` (path, required)
  - `taskId` (path, required)
  - `body` (body, required)
- **Purpose**: Add comment to task

---

### Time Sheets

#### addManualTime
- **Method**: POST
- **Endpoint**: `/businesses/{businessId}/jobs/{jobId}/timesheets/manual`
- **Parameters**:
  - `businessId` (path, required)
  - `jobId` (path, required)
  - `body` (body, required)
- **Purpose**: Manually add time entry

#### getTimeSheets
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/jobs/{jobId}/timesheets`
- **Parameters**:
  - `businessId` (path, required)
  - `jobId` (path, required)
- **Purpose**: Retrieve job timesheets

#### updateTimeSheet
- **Method**: PATCH
- **Endpoint**: `/businesses/{businessId}/jobs/{jobId}/timesheets/{timeSheetId}`
- **Parameters**:
  - `businessId` (path, required)
  - `jobId` (path, required)
  - `timeSheetId` (path, required)
  - `body` (body, required)
- **Purpose**: Modify timesheet entry

#### deleteTimeSheet
- **Method**: DELETE
- **Endpoint**: `/businesses/{businessId}/jobs/{jobId}/timesheets/{timeSheetId}`
- **Parameters**:
  - `businessId` (path, required)
  - `jobId` (path, required)
  - `timeSheetId` (path, required)
- **Purpose**: Remove timesheet entry

#### clockIn
- **Method**: POST
- **Endpoint**: `/businesses/{businessId}/jobs/{jobId}/clock-in`
- **Parameters**:
  - `businessId` (path, required)
  - `jobId` (path, required)
- **Purpose**: Start time tracking

#### clockOut
- **Method**: POST
- **Endpoint**: `/businesses/{businessId}/jobs/{jobId}/timesheets/{timeSheetId}/clock-out`
- **Parameters**:
  - `businessId` (path, required)
  - `jobId` (path, required)
  - `timeSheetId` (path, required)
- **Purpose**: End time tracking

#### getActiveTimeSheet
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/jobs/{jobId}/timesheets/active`
- **Parameters**:
  - `businessId` (path, required)
  - `jobId` (path, required)
- **Purpose**: Get currently active timesheet

---

### Orders

#### fetchPriceSummary
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/price-summary`
- **Parameters**:
  - `businessId` (path, required)
  - `customerId` (query, required)
  - `addressId` (query, required)
  - `paymentPlanType` (query, required)
  - `packageIds` (query, required)
- **Purpose**: Calculate order pricing

#### placeRecurringJobOrder
- **Method**: POST
- **Endpoint**: `/businesses/{businessId}/customers/{customerId}/recurring-orders`
- **Parameters**:
  - `businessId` (path, required)
  - `customerId` (path, required)
  - `request` (body, required)
- **Purpose**: Create recurring service order

#### placeOneTimeJobOrder
- **Method**: POST
- **Endpoint**: `/businesses/{businessId}/customers/{customerId}/one-time-orders`
- **Parameters**:
  - `businessId` (path, required)
  - `customerId` (path, required)
  - `request` (body, required)
- **Purpose**: Create one-time service order

---

### Crews

#### fetchCrews
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/crews`
- **Parameters**:
  - `businessId` (path, required)
- **Purpose**: List available crews

---

### Payment

#### proceedPayment
- **Method**: POST
- **Endpoint**: `/businesses/{businessId}/payment`
- **Parameters**:
  - `businessId` (path, required)
  - `body` (body, required)
- **Purpose**: Process payment transaction

#### checkPaymentStatus
- **Method**: GET
- **Endpoint**: `/payment-status`
- **Parameters**:
  - `paymentStatusId` (query, required)
- **Purpose**: Verify payment status

---

### Visits

#### fetchVisits
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/visits`
- **Parameters**:
  - `businessId` (path, required)
  - `page` (query, optional)
  - `limit` (query, optional)
  - `search` (query, optional)
- **Purpose**: List visits with filtering

#### updateVisit
- **Method**: PATCH
- **Endpoint**: `/businesses/{businessId}/visits/{id}`
- **Parameters**:
  - `businessId` (path, required)
  - `id` (path, required)
  - `body` (body, required)
- **Purpose**: Update visit details

#### getVisitDetails
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/visits/{id}`
- **Parameters**:
  - `businessId` (path, required)
  - `id` (path, required)
- **Purpose**: Get detailed visit information

---

### Leads

#### getLeads
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/leads`
- **Parameters**:
  - `businessId` (path, required)
  - `page` (query, optional)
  - `limit` (query, optional)
  - `search` (query, optional)
- **Purpose**: List leads with filtering

#### createLead
- **Method**: POST
- **Endpoint**: `/businesses/{businessId}/leads`
- **Parameters**:
  - `businessId` (path, required)
  - `body` (body, required)
- **Purpose**: Create new lead

#### getLeadDetails
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/leads/{leadId}`
- **Parameters**:
  - `businessId` (path, required)
  - `leadId` (path, required)
- **Purpose**: Get detailed lead information

#### updateLeadDetails
- **Method**: PATCH
- **Endpoint**: `/businesses/{businessId}/leads/{leadId}`
- **Parameters**:
  - `businessId` (path, required)
  - `leadId` (path, required)
  - `body` (body, required)
- **Purpose**: Update lead details

---

### Tasks

#### fetchTasks
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/tasks`
- **Parameters**:
  - `businessId` (path, required)
  - `search` (query, optional)
  - `page` (query, optional)
  - `limit` (query, optional)
- **Purpose**: List tasks with filtering

#### createTask
- **Method**: POST
- **Endpoint**: `/businesses/{businessId}/tasks`
- **Parameters**:
  - `businessId` (path, required)
  - `body` (body, required)
- **Purpose**: Create new task

#### getTaskDetails
- **Method**: GET
- **Endpoint**: `/businesses/{businessId}/tasks/{taskId}`
- **Parameters**:
  - `businessId` (path, required)
  - `taskId` (path, required)
- **Purpose**: Get detailed task information

#### updateTaskDetails
- **Method**: PATCH
- **Endpoint**: `/businesses/{businessId}/tasks/{taskId}`
- **Parameters**:
  - `businessId` (path, required)
  - `taskId` (path, required)
  - `body` (body, required)
- **Purpose**: Update task details

---

### AI Integration

#### getAIGeneratedContent
- **Method**: POST
- **Endpoint**: `/ai/interact`
- **Parameters**:
  - `body` (body, required)
- **Purpose**: Interact with AI for content generation

---

## Common Patterns

### Pagination
Most list endpoints support pagination:
- `page` (query): Page number (optional)
- `limit` (query): Items per page (optional)

### Search
Many endpoints support search functionality:
- `search` (query): Search term (optional)

### Business Context
Almost all endpoints require `businessId` as a path parameter for multi-tenant support.

### Response Type
All methods return `Future<HttpResponse>` for consistent response handling with status codes and data.
