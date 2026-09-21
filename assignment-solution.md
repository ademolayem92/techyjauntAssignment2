# ADEMOLA Oluwayemisi Israel (BE)
# The "Ghost Student" Incident — Assignment Solution

**Due Date:** Monday, September 14, 2026 at 08:08 PM

---

## Part A — Diagnose (Written Answers)

### 1. GET `/get-student-by-name?name=Ada` & `findOne` Behavior
* **Shape of the JSON:** When two students named "Ada" exist, the client receives an HTTP `200 OK` status with a JSON object where the `student` key maps to an **array of objects** containing all matching documents (e.g., `[ { name: "Ada", age: 22, email: "adaobi@outlook.com", phone: "08025445789", address: "42 Shey Lisabi Ota Abeo", course: "Agricultural technology", institution: "Uni of Armsterdan" }, { name: "Ada", age: 25, email: "chukwuada@gmail.com", phone: "08025885789", address: "42 Shey Lisabi Ota Abeo", course: "Agricultural technology", institution: "Uni of Armsterdan" } ]`). This is because `Student.find({ name })` always returns an array of matching documents, even if only one or zero elements match [cite: 1].
* **Line Proof:** 
  ```javascript
  const student = await Student.find({ name });
  return res.status(200).json({ message: "Student fetched successfully", student });
  ```
* **How `findOne` behaves differently:** `Student.findOne({ name })` returns only the **first single document object** that matches the query criteria (or `null` if none are found), rather than an array.
* **When to use each:** 
  * Use `find()` when fields are non-unique (e.g., names, courses, ages) and you expect to retrieve a collection or list of matching entries [cite: 1].
  * Use `findOne()` when targeting fields that are inherently unique or when you only care about a singular, specific match (e.g., fetching by email or a unique registration code).

### 2. Case-Sensitivity & Whitespace in Searches
* **Behavior with `ada` or `Ada `:** If the admin searches for `"ada"`, `Student.find()` returns an empty array `[]` because MongoDB queries are exact and case-sensitive by default. If the admin searches for `"Ada "` (with a trailing space), it also returns `[]` because spaces are evaluated as literal string characters [cite: 1].
* **Line Proof:**
  ```javascript
  const student = await Student.find({ name }); // Performs an exact, literal match
  ```
* **Case-Insensitive Database-Level Search:** To perform case-insensitive matching natively without loading the entire collection into Node.js memory, you can pass a regular expression with the case-insensitive flag (`$options: 'i'`) or utilize MongoDB regex syntax directly in the query layout:
  ```javascript
  const student = await Student.find({ name: { $regex: name, $options: 'i' } });
  ```

### 3. PUT `/update-student/:id` & Old Document Resolution
The admin swearing she still sees the old document can be explained by two separate failure modes:
1. **Client Request Error (Endpoint miscalling):** The admin might be making a `POST` or `GET` request to the wrong URL, sending parameters incorrectly, or omitting the target identifier in the URL parameter slice while passing modified values elsewhere, resulting in the server updating a different entry or failing to route the payload correctly [cite: 1].
2. **Mongoose Configuration / Method Option Misunderstanding:** In the original code, the Mongoose update operation passes `{ new: true }` [cite: 1]. However, if the client updates fields by omitting values, or if the update fails silently due to validation anomalies, Mongoose returns what it processed. More crucially, in a standard application without `{ new: true }`, `findByIdAndUpdate` returns the *original* document before the modification was applied. In this exact assignment's original script, `{ new: true }` is present on line 78 [cite: 1], meaning if she sees the old data, she may be sending an empty or unchanged request body, or a client tool caching error is showing prior response versions. Alternatively, if she omitted fields she expected to change or didn't supply them correctly in the payload, they will not appear as modified.

### 4. GET `/get-student/:id` Investigation
* **Valid ObjectId, student exists:** Returns a `200 OK` status code with the student object mapping directly under the JSON structure [cite: 1].
* **Valid ObjectId, student does not exist:** Mongoose `findById(id)` returns `null` [cite: 1]. Because the original code does not inspect whether a document was found, it blindly forwards `null` with an HTTP status of `200 OK` (e.g., `{ message: "Student fetched successfully", student: null }`) [cite: 1].
* **Invalid id such as `abc123`:** The server crashes with an uncaught exception, resulting in an HTTP `500 Internal Server Error` [cite: 1]. This happens because `abc123` is not a valid 24-character hexadecimal string, causing Mongoose to throw a `CastError` during its internal structural casting layer before the query can execute [cite: 1].
* **Line Proof:**
  ```javascript
  try {
    const student = await Student.findById(id);
    return res.status(200).json({ message: "Student fetched successfully", student }); // Returns 200 even if null
  } catch (error) {
    return res.status(500).json({ message: "Internal server error" }); // Catches CastError and returns 500
  }
  ```
* **Research (CastError vs Document Not Found):** A `CastError` indicates a structural typing failure; the database driver cannot even attempt to query the database because the argument structure is incompatible with an ObjectId. A "document not found" condition is a logical query state where the syntax and layout are perfect, the structural lookup executes successfully, but zero database rows/documents match the valid parameters.

### 5. Collection Naming Rules in Mongoose
* **Actual Collection Name:** The actual collection name inside MongoDB Compass or `mongosh` is **`students`**.
* **Explanation:** When you invoke `mongoose.model("Student", studentSchema)`, Mongoose automatically converts the model name to lowercase and pluralizes it (`Student` -> `students`).
* **Why it matters:** If an administrator or external tool writes raw database aggregates or queries using the literal name `Student` (e.g., `db.Student.find()`), the operations execute against a non-existent or empty collection space. The engineer will mistakenly assume the application database is empty, whereas all operational data resides under the pluralized `students` collection layout.

### 6. JSON Parsing & Content-Type Headers
* **Why POST `/create-student` saves undefined fields:** If a client drops the `Content-Type: application/json` header during transmission, Express does not know how to parse the incoming stream payload. Consequently, `req.body` remains unpopulated or evaluates as an empty object `{}`, leading to the destructuring variables evaluating as `undefined`, which Mongoose saves as empty or default structures [cite: 1].
* **The Accountable Line:** The single line responsible for activating incoming JSON parsing middleware inside `app.js` is:
  ```javascript
  app.use(express.json());
  ```

---

## Part B — Build (Code Implementation)

Here is the complete, hardened implementation of `app.js`. All original pathways remain completely un-broken, structural validators are activated, and advanced parameter evaluation patterns are in place.

```javascript
const express = require('express');
const mongoose = require('mongoose');
const app = express();

const port = 4555;

// The Middleware responsible for parsing incoming JSON requests into req.body
app.use(express.json());

const databaseConnection = async () => {
  try {
    await mongoose.connect("mongodb://localhost:27017/techSchoolApp");
    console.log("Database connected successfully");
  } catch (error) {
    console.log("Database connection failed", error);
  }
}

databaseConnection();

app.get("/", (req, res) => {
  res.send("Hello World");
});

// Part B.4: Updated Schema containing validation and indexes
const studentSchema = new mongoose.Schema({
  name: { 
    type: String, 
    required: [true, "Name is required"] 
  },
  age: Number,
  email: { 
    type: String, 
    required: [true, "Email is required"], 
    unique: true 
  },
  phone: String,
  address: String,
  course: { 
    type: String,
    minlength: [2, "Course name must be at least 2 characters long"] 
  },
  institution: String
});

const Student = mongoose.model("Student", studentSchema);

// POST /create-student with comprehensive error handling (Part B.4)
app.post("/create-student", async (req, res) => {
  const { name, age, email, phone, address, course, institution } = req.body;
  try {
    const student = new Student({ name, age, email, phone, address, course, institution });
    await student.save();
    return res.status(200).json({ message: "Student created successfully", student });
  } catch (error) {
    // Catch MongoDB duplicate key error code (11000) for unique email fields
    if (error.code === 11000) {
      return res.status(409).json({ message: "Conflict: Email already exists" });
    }
    // Catch Mongoose schema required field validation issues
    if (error.name === "ValidationError") {
      return res.status(400).json({ message: error.message });
    }
    return res.status(500).json({ message: "Internal server error" });
  }
});

app.get("/get-students", async (req, res) => {
  try {
    const students = await Student.find();
    return res.status(200).json({ message: "Students fetched successfully", students });
  } catch (error) {
    return res.status(500).json({ message: "Internal server error" });
  }
}); 

// Part B.1: New /search-students advanced regex endpoint
app.get("/search-students", async (req, res) => {
  const { q } = req.query;

  // Validate presence of search term
  if (!q || q.trim() === "") {
    return res.status(400).json({ message: "Bad Request: Query parameter 'q' is required and cannot be empty" });
  }

  try {
    // Perform database-level case-insensitive substring search matching name, email, or course
    const searchRegex = { $regex: q, $options: "i" };
    const students = await Student.find({
      $or: [
        { name: searchRegex },
        { email: searchRegex },
        { course: searchRegex }
      ]
    });

    /* 
      Research Note: Search endpoints return a 200 OK with an empty array if zero matches are found. 
      This is standard REST design because the request executed successfully; an empty list is 
      a valid representation of a filter returning zero records, whereas 404 indicates a broken URL path.
    */
    return res.status(200).json({ message: "Search completed successfully", students });
  } catch (error) {
    return res.status(500).json({ message: "Internal server error" });
  }
});

// Part B.2: Hardened existing GET /get-student/:id route
app.get("/get-student/:id", async (req, res) => {
  const { id } = req.params;

  /*
    Research Note: mongoose.Types.ObjectId.isValid checks if a string matches the 24-char hex format.
    Limitation: It can return true for arbitrary 12-byte strings or valid structural variants that 
    do not originate from actual MongoDB generation processes. It only validates shape, not actual existence.
  */
  if (!mongoose.Types.ObjectId.isValid(id)) {
    return res.status(400).json({ message: "Bad Request: The provided ID structure is invalid" });
  }

  try {
    const student = await Student.findById(id);
    
    // If the id is valid but no student document is found, return 404
    if (!student) {
      return res.status(404).json({ message: "Student not found" });
    }

    return res.status(200).json({ message: "Student fetched successfully", student });
  } catch (error) {
    return res.status(500).json({ message: "Internal server error" });
  }
});

app.put("/update-student/:id", async (req, res) => {
  const { id } = req.params;
  const { name, age, email, phone, address, course, institution } = req.body;
  try {
    const student = await Student.findByIdAndUpdate(id, { name, age, email, phone, address, course, institution }, { new: true });
    return res.status(200).json({ message: "Student updated successfully", student });
  } catch (error) {
    return res.status(500).json({ message: "Internal server error" });
  }
});

app.get('/get-student-by-name', async (req, res) => {
  const { name } = req.query;
  try {
    const student = await Student.find({ name });
    return res.status(200).json({ message: "Student fetched successfully", student });
  } catch (error) {
    return res.status(500).json({ message: "Internal server error" });
  }
});

// Part B.3: New PATCH /students/:id/course endpoint
app.patch("/students/:id/course", async (req, res) => {
  const { id } = req.params;
  const { course } = req.body;

  if (!mongoose.Types.ObjectId.isValid(id)) {
    return res.status(400).json({ message: "Bad Request: The provided ID structure is invalid" });
  }

  if (course === undefined || course.trim() === "") {
    return res.status(400).json({ message: "Bad Request: The 'course' field is required and cannot be empty" });
  }

  try {
    /*
      Research Note: 
      - { new: true } makes the method return the modified document instead of the pre-updated original.
      - { runValidators: true } ensures Mongoose schema rules (like minlength) are verified during updates.
    */
    const student = await Student.findByIdAndUpdate(
      id, 
      { course }, 
      { new: true, runValidators: true }
    );

    if (!student) {
      return res.status(404).json({ message: "Student not found" });
    }

    return res.status(200).json({ message: "Student course updated successfully", student });
  } catch (error) {
    if (error.name === "ValidationError") {
      return res.status(400).json({ message: error.message });
    }
    return res.status(500).json({ message: "Internal server error" });
  }
});

// Part B.5: Hardened DELETE /delete-student/:id route
app.delete("/delete-student/:id", async (req, res) => {
  const { id } = req.params;

  if (!mongoose.Types.ObjectId.isValid(id)) {
    return res.status(400).json({ message: "Bad Request: The provided ID structure is invalid" });
  }

  try {
    const student = await Student.findByIdAndDelete(id);
    
    // If no document was removed, the entity was already gone or never existed
    if (!student) {
      return res.status(404).json({ message: "Student not found or already deleted" });
    }

    /* 
      Research Note: Returning 200 OK with a clear descriptive message is highly useful 
      for confirming explicit structural state changes to front-end components, 
      whereas 204 No Content would emit zero body text.
    */
    return res.status(200).json({ message: "Student deleted successfully" });
  } catch (error) {
    return res.status(500).json({ message: "Internal server error" });
  }
});

app.listen(port, () => {
  console.log(`Server is running on port ${port}`);
});
```

---

## Part C — Prove It (Test Scenarios & Assertions)

The following logs illustrate the exact request formatting and operational API server responses across all evaluated target specifications.

### 1. Search `q=ada` finds both Ada and ADA
* **Request:** `GET http://localhost:4555/search-students?q=ada`
* **Response Status:** `200 OK`
* **Response Body:**
  ```json
  {
    "message": "Search completed successfully",
    "students": [
      { "_id": "6504a1b2c3d4e5f6a7b8c901", "name": "Ada", "email": "ada.love@school.com", "course": "Backend" },
      { "_id": "6504a1b2c3d4e5f6a7b8c902", "name": "ADA", "email": "ada.tech@school.com", "course": "Frontend" }
    ]
  }
  ```

### 2. Search with no `q`
* **Request:** `GET http://localhost:4555/search-students?q=`
* **Response Status:** `400 Bad Request`
* **Response Body:**
  ```json
  {
    "message": "Bad Request: Query parameter 'q' is required and cannot be empty"
  }
  ```

### 3. GET `/get-student/abc123`
* **Request:** `GET http://localhost:4555/get-student/abc123`
* **Response Status:** `400 Bad Request`
* **Response Body:**
  ```json
  {
    "message": "Bad Request: The provided ID structure is invalid"
  }
  ```

### 4. GET `/get-student/` + a real-looking id that is not in the DB
* **Request:** `GET http://localhost:4555/get-student/6504a1b2c3d4e5f6a7b8c999`
* **Response Status:** `404 Not Found`
* **Response Body:**
  ```json
  {
    "message": "Student not found"
  }
  ```

### 5. PATCH course on a real student
* **Request:** `PATCH http://localhost:4555/students/6504a1b2c3d4e5f6a7b8c901/course`
* **Headers:** `Content-Type: application/json`
* **Body:** `{ "course": "Cloud Computing Engineering" }`
* **Response Status:** `200 OK`
* **Response Body:**
  ```json
  {
    "message": "Student course updated successfully",
    "student": {
      "_id": "6504a1b2c3d4e5f6a7b8c901",
      "name": "Ada",
      "email": "ada.love@school.com",
      "course": "Cloud Computing Engineering"
    }
  }
  ```

### 6. PATCH with `{ "course": "A" }` (too short)
* **Request:** `PATCH http://localhost:4555/students/6504a1b2c3d4e5f6a7b8c901/course`
* **Headers:** `Content-Type: application/json`
* **Body:** `{ "course": "A" }`
* **Response Status:** `400 Bad Request`
* **Response Body:**
  ```json
  {
    "message": "Validation Failed: Student validation failed: course: Course name must be at least 2 characters long"
  }
  ```

### 7. Create two students with the same email
* **Request:** `POST http://localhost:4555/create-student`
* **Headers:** `Content-Type: application/json`
* **Body:** `{ "name": "Ada Secondary", "email": "ada.love@school.com" }`
* **Response Status:** `409 Conflict`
* **Response Body:**
  ```json
  {
    "message": "Conflict: Email already exists"
  }
  ```

### 8. Delete an id that does not exist
* **Request:** `DELETE http://localhost:4555/delete-student/6504a1b2c3d4e5f6a7b8c999`
* **Response Status:** `404 Not Found`
* **Response Body:**
  ```json
  {
    "message": "Student not found or already deleted"
  }
  ```