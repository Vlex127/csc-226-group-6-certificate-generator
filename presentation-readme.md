# Presentation Guide — LASU Certificate Generator

**Speaker Notes for CSC 226 Presentation**

Slide-by-slide walkthrough with technical explanations, talking points, and tips for fielding questions.

---

## Slide 1: Title Slide

### What to Say
"Good morning/afternoon. We are Group 6 from CSC 226 — Object-Oriented Programming I. Today we present our project: the **LASU Certificate Generator**, a production-grade C++ web application that demonstrates all the OOP concepts we've covered this semester."

### Key Points to Hit
- "Built entirely in C++17 with zero external dependencies"
- "Runs its own HTTP server using Winsock2 — no Apache, no IIS, no Node.js"
- "Generates printable HTML certificates with three different visual styles"
- "Implements 8+ design patterns including Strategy, Template Method, Factory, and more"

### Group Introduction
Have 1-2 group members briefly introduce themselves, then say "the full list of 20 members is in the project README."

---

## Slide 2: Overview & Architecture

### What to Say
"This slide shows the high-level architecture. At the top, the user's browser makes HTTP requests to our custom C++ web server running on localhost:8080. The server delegates to the CertificateGenerator which manages student data in a `std::map` and uses the Certificate + Template hierarchy to generate HTML output."

### Architecture Walkthrough
1. **Browser** — The frontend is a single-page application (SPA) with HTML/CSS/JS embedded directly in the C++ source code as a raw string literal
2. **HttpServer** — Custom Winsock2 TCP server. Each incoming connection gets its own detached thread
3. **CertificateGenerator** — Central data manager using `std::map<std::string, Student>` for O(log n) lookups. All operations are thread-safe via `std::mutex`
4. **Certificate + Template** — Where the design patterns live. Strategy pattern for interchangeable visual styles, Template Method for the certificate generation algorithm

### Data Flow
"Data flows one way: Browser → HTTP → Server → Generator → Template → HTML file. The browser never speaks directly to the generator — everything goes through the REST API."

### Anticipated Questions
- *Q: Why C++ and not a web framework?* A: "The assignment required demonstrating OOP in C++. Building our own HTTP server actually gave us more opportunities to show patterns than using a framework would have."
- *Q: How many concurrent users?* A: "It's designed for single-user desktop use, but the thread-per-connection model could handle dozens of concurrent requests."

---

## Slide 3: Inheritance & Encapsulation

### What to Say
"This slide demonstrates two foundational OOP concepts. **Inheritance** is shown through the `Person` → `Student` hierarchy. `Person` is an abstract base class with a pure virtual `getDescription()` method and a protected `name_` field. `Student` inherits from `Person`, adds `course_` and `grade_`, and implements `getDescription()`."

### Encapsulation
"**Encapsulation** is visible in the `CertificateGenerator` class. All data — the `students_` map and the `mutex_` — is private. External code can only interact through public methods like `addStudent()` which validates input before storing. Bad data throws a `ValidationException`."

### Code Highlights
- "The `protected` keyword in Person allows derived classes to access `name_` while keeping it hidden from the outside world"
- "`isValidGrade()` is a static method — it belongs to the class, not any instance. We use it to validate grades before constructing a Student object"
- "The `gradeToPoints()` method converts both letter grades (A, B) and numeric grades (0-100) to the 4.0 GPA scale"

### Anticipated Questions
- *Q: Why use `std::map` instead of `std::vector`?* A: "We need fast lookups by name. With a map, finding a student is O(log n). With a vector, we would need to scan the entire list."
- *Q: Why is `mutex_` marked `mutable`?* A: "Because `findStudent()` and `getAllStudents()` are `const` methods but still need to lock the mutex for thread safety."

---

## Slide 4: Polymorphism & Abstract Base Classes

### What to Say
"Our project has **three abstract base classes** with a total of **seven pure virtual methods**. This slide focuses on the `Certificate` hierarchy."

### Explain
- `Certificate` is an abstract base with `generateHTML()` (the Template Method) and `getTypeName()` (pure virtual)
- Three concrete subclasses: `CertificateOfExcellence`, `CertificateOfCompletion`, `CertificateOfParticipation`
- "At runtime, the factory function creates the correct subclass based on an enum value. The client code works through base class pointers — it never needs to know which concrete type it's dealing with."

### Polymorphism Demo
"This is the key moment: when we call `cert->generateHTML()`, C++ looks up the vtable at runtime and dispatches to the right method. The same line of code produces different behavior depending on the concrete type."

### Anticipated Questions
- *Q: What are the 7 pure virtual methods?* A: "`getDescription()` in Person, `header()/styles()/body()` in CertificateTemplate, `getTypeName()/getDescriptor()` in Certificate, and `generateHTML()` in Certificate (which is virtual but has a default implementation)."

---

## Slide 5: Strategy + Template Method

### What to Say
"This is the heart of our design. We combine two patterns — **Strategy** and **Template Method** — to get maximum code reuse with maximum flexibility."

### Strategy Pattern
"The `CertificateTemplate` interface defines three methods: `header()`, `styles()`, and `body()`. Each concrete template — Formal, Modern, Minimalist — provides its own implementation. The Formal template uses a serif font with a double border. Modern uses a purple gradient background with rounded corners. Minimalist keeps it clean with monospace and thin borders."

### Template Method Pattern
"The `generateHTML()` method in `Certificate` defines the skeleton algorithm: it calls `header()`, then `styles()`, then `body()` in that fixed order. Subclasses cannot change this order — they only override `getTypeName()` to plug their specific content into the right step."

### Why Combine Them?
"Strategy handles the *visual variation* — swapping the entire look and feel. Template Method handles the *content variation* — what type of certificate it is. They are orthogonal concerns, so we separate them into two class hierarchies."

### Anticipated Questions
- *Q: Could you add a new template?* A: "Yes. Create a new class that inherits from `CertificateTemplate`, implement the three methods, add a case to the factory's switch statement. That's it — zero changes to any existing code. This is the Open/Closed Principle in action."
- *Q: How is Template Method different from just overriding?* A: "Template Method *controls the structure*. The base class decides when each step runs. Subclasses only fill in the blanks. With regular overriding, a subclass could completely change the algorithm."

---

## Slide 6: Factory + Composition

### What to Say
"The **Factory pattern** centralizes object creation. Instead of scattering `make_unique<FormalTemplate>()` all over the codebase, we have two factory functions: `makeTemplate()` and `makeCertificate()`. You pass an enum, you get back the right polymorphic type."

### Factory Example
```cpp
auto tmpl = makeTemplate(Formal);      // Returns a FormalTemplate
auto cert = makeCertificate(Excellence, student, std::move(tmpl));
```

### Composition
"**Composition** means 'has-a' instead of 'is-a'. A Certificate *has-a* CertificateTemplate — it doesn't *inherit from* a template. This gives us runtime flexibility. We could swap the template of an existing certificate if we wanted to. It also means looser coupling — changes to one class don't force changes on the other."

### Chain of Composition
"The whole project is built on composition chains: `HttpServer` has-a `CertificateGenerator`, which has-a `map` of `Student`s. When generating, `Certificate` has-a `CertificateTemplate`."

### Anticipated Questions
- *Q: Why prefer composition over inheritance?* A: "Composition is more flexible (runtime-swappable), more testable (you can mock components), and avoids the fragile base class problem."
- *Q: Why `std::unique_ptr` in the factory?* A: "It communicates ownership clearly — the caller now owns the created object. It's also RAII — the object is automatically deleted when it goes out of scope."

---

## Slide 7: RAII & STL Features

### What to Say
"**RAII — Resource Acquisition Is Initialization** — is the most important C++ concept in our project. Resources are acquired during object construction and released during destruction. You never see a manual `delete`, `close()`, or `unlock()` in our code."

### RAII Examples
- `std::lock_guard<std::mutex>` — locks automatically on construction, unlocks on destruction (even if an exception is thrown)
- `std::unique_ptr` — auto-deletes the polymorphic Certificate and Template objects when they go out of scope
- `std::ofstream` — auto-closes the file when the stream is destroyed

### STL Features
- `std::map` — O(log n) sorted storage, used for the student database
- `std::find_if` + lambdas — used for partial name matching in search
- `std::sort` + lambdas — used for sorting by name, grade, or course
- `std::thread` — each HTTP connection gets its own thread
- `std::mutex` — thread safety for shared data

### Code Demo Focus
"In `addStudent()`, we create a `lock_guard` at the top. Even if the function throws an exception halfway through, the `lock_guard` destructor runs and the mutex is unlocked. This is impossible to achieve with manual lock/unlock without leaking."

---

## Slide 8: HTTP Server & REST API

### What to Say
"This slide show how we handle network communication. Our HTTP server is built from scratch using Winsock2 — Windows' socket API. It's only about 670 lines but it serves a complete web application."

### Architecture
- Server listens on port 8080
- Accepts connections in an infinite loop
- Each connection gets its own thread (`std::thread(...).detach()`)
- The thread parses the raw HTTP request, routes it, and sends back a JSON or HTML response

### API Endpoints
"We have 12 endpoints. The main ones are: `GET /` serves the web UI, `GET /list` returns all students as JSON, `POST /add` inserts a student, `POST /generate` creates certificates, `GET /stats` returns computed analytics."

### GPA Calculation
"The average GPA you see on the Dashboard and Statistics tabs is computed in three steps:"

1. **Grade to Points Conversion** — Each student's grade string is converted to a 4.0-scale value using a static lookup table:
   ```
    A → 5.0 ,  B → 4.0  , C → 3.0
          D → 2.0 , E → 1.0  , F → 0.0
   ```
   "If the grade is a number like '85', we scale it to the 5.0 scale by dividing by 20 (`85 / 20 = 4.25`)."

2. **Summation** — All students' GPA points are added together in `computeStats()`:
   ```cpp
   double totalPoints = 0;
   for (const auto& [name, student] : students_) {
       totalPoints += student.gradeToPoints();
   }
   ```

3. **Division** — The total is divided by the number of students:
   ```cpp
   stats.avgGradePoints = totalPoints / students_.size();
   ```

"Both letter grades (A, B, etc.) and numeric grades (0-100) are supported. The `isValidGrade()` static method validates before any grade is stored — invalid grades throw a `ValidationException`."

### Frontend
"The entire frontend — HTML, CSS, and JavaScript — is embedded in the C++ source code as a raw string literal. The JavaScript uses the Fetch API to make AJAX calls to our backend. No page reloads, no frameworks."

### Thread Safety
"All generator methods are protected by `std::mutex`. Even though this is a single-user application, the thread-per-connection model means we could have concurrent requests. The mutex ensures data integrity."

---

## Slide 9: Web UI Preview

### What to Say
[Point to screenshot]
"This is what the application looks like in the browser. The UI has four main tabs:"

1. **Dashboard** — Shows total students, average GPA, certificate count, and active courses. Quick action buttons for common tasks.
2. **Students** — Full data table with search (by name or course), add/remove functionality, and grade badges with color coding.
3. **Generate** — Dropdowns for certificate type and template style. Generate for one student or all. Generated files are listed with clickable links.
4. **Statistics** — Course distribution and grade distribution breakdowns.

### UI Features to Highlight
- "Modern dark/light theme with system-aware toggle"
- "Toast notifications for success and error feedback"
- "Responsive layout that works on different screen sizes"
- "All AJAX — no page reloads for any operation"

---

## Slide 10: All Concepts in Action

### What to Say
"Now let's see everything working together in one complete example."

### Walk Through the Code
1. **Create a Student** — Encapsulation: we call `addStudent()` which validates before storing. If the grade is invalid, it throws `ValidationException`.
2. **Create a Template** — Factory: `makeTemplate(Formal)` returns a `FormalTemplate` (but the caller only sees the base class).
3. **Create a Certificate** — Factory + Composition: `makeCertificate(Excellence, student, tmpl)` returns a `CertificateOfExcellence` that *has a* FormalTemplate.
4. **Generate HTML** — Template Method + Polymorphism: `cert->generateHTML()` dispatches polymorphically. The Template Method calls `getTypeName()` → "Excellence", then calls the template's header, styles, and body in order.
5. **Write to File** — RAII: `ofstream` auto-closes when it goes out of scope.
6. **Serve via HTTP** — Thread safety: the mutex-protected generator is accessed from a detached thread.

### Concept Grid
[Point to the concept cards at the bottom]
"These 12 cards summarize every OOP concept demonstrated in this flow. Each concept has a real code example in our project."

---

## Slide 11: Thank You

### What to Say
"Thank you for your attention. We're happy to take any questions."

### Closing Points
- "All source code and documentation is available in our project repository"
- "The full README includes setup instructions, API reference, and complete group member list"
- "We welcome feedback and suggestions for future improvements"

### If Asked About Future Work
- "We would like to add PDF export using a C++ PDF library"
- "Database persistence instead of the current in-memory `std::map`"
- "Authentication system for multi-user support"
- "More template styles — perhaps department-specific templates"

---

## General Presentation Tips

### Timing
- Slides 1-2: 3 minutes
- Slides 3-7 (pattern deep-dives): 2 minutes each = 10 minutes
- Slide 8-9 (HTTP + UI): 3 minutes
- Slide 10 (integration): 3 minutes
- Slide 11 (closing): 1 minute
- **Total: ~20 minutes + Q&A**

### Who Presents What
| Slides | Presenter | Focus |
|---|---|---|
| 1-2 | Member 1 | Introduction, architecture overview |
| 3-4 | Member 2 | Inheritance, polymorphism, encapsulation |
| 5-6 | Member 3 | Strategy, Template Method, Factory, Composition |
| 7-8 | Member 4 | RAII, STL, HTTP Server |
| 9-10 | Member 5 | UI demo, integration walkthrough |
| 11 | Any | Thank you + Q&A |

### Hot Keys (HTML Presentation)
- `→` / `Space`: Next slide
- `←`: Previous slide
- `F`: Full screen
- `Esc`: Overview
- `S`: Speaker notes (if configured)

### Common Questions to Prepare For
1. "Why did you build your own HTTP server instead of using a library?"
2. "How is this different from just printing HTML files?"
3. "What would you change if you had more time?"
4. "How did you split work among 20 group members?"
5. "Can you explain virtual functions and the vtable?"
6. "Why `std::map` and not `std::unordered_map`?"
