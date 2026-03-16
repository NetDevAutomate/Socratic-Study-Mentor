# Swift/SwiftUI PoC Feasibility Study: Flashcard App with Claude API

**Date**: 2026-03-15
**Context**: Evaluating whether a Python developer (experienced, new to Swift) can build a macOS SwiftUI flashcard app PoC with PDF extraction, Claude API integration, SQLite storage, and localhost server for future MCP integration.

---

## Executive Summary

**Verdict: Feasible, with caveats.** The PoC is achievable in 2-4 weeks with AI pair programming. The biggest risks are not the individual components (all well-supported) but the compound learning curve of Swift's type system + SwiftUI's declarative paradigm + Xcode's tooling, all at once. The ecosystem has every package you need. The question is whether this is the *right* tool for the job vs. extending the existing Python/web stack.

---

## 1. SwiftUI Learning Curve for a Python Developer

### Honest Assessment: Moderate-to-Steep

SwiftUI is approachable for simple UIs but becomes confusing fast when you hit its sharp edges. Here are the paradigm shifts, ranked by difficulty:

### Paradigm Shifts (Hardest First)

| Shift | Python Equivalent | Swift Reality | Pain Level |
|-------|------------------|---------------|------------|
| **Type system strictness** | Duck typing, `Any` | Generics, protocols, associated types, `some View` opaque types | HIGH |
| **Value vs Reference semantics** | Everything is a reference | `struct` (value, copied) vs `class` (reference, shared). SwiftUI views are ALL structs. | HIGH |
| **Reactive/Declarative UI** | Imperative (Textual is closest) | Views are a *function of state*. You never mutate the UI directly. | MEDIUM-HIGH |
| **Property wrappers for state** | No equivalent | `@State`, `@Binding`, `@Observable`, `@Environment` -- each has specific rules about ownership and scope | MEDIUM-HIGH |
| **Optionals everywhere** | `None` with duck typing | `String?` vs `String` are different types. Force-unwrapping (`!`) crashes. Must use `if let`, `guard let`, or `??` | MEDIUM |
| **Closures as first-class** | Lambdas, but limited | Trailing closure syntax, `@escaping`, capture lists `[weak self]` | MEDIUM |
| **Protocol-oriented design** | ABC/Protocols (PEP 544) | Protocols are central. `Codable`, `Identifiable`, `Hashable` conformance required constantly | MEDIUM |
| **Swift concurrency** | `asyncio` | `async/await` + `Task` + `@MainActor` + actors. Similar concepts, different execution model. | LOW-MEDIUM |
| **Error handling** | `try/except` | `do { try } catch` -- similar pattern, less flexible (no bare `except`) | LOW |

### The Biggest Traps

1. **"Cannot assign to property: 'self' is immutable"** -- SwiftUI views are structs. You cannot mutate properties directly. Must use `@State` or `@Binding`. This will confuse you repeatedly in week 1.

2. **Opaque return types (`some View`)** -- Every SwiftUI view body returns `some View`. The compiler infers the concrete type. When you get type errors, the messages are *atrocious* and point to the wrong line.

3. **SwiftUI view identity and lifecycle** -- Views are recreated constantly. State must live outside the view in `@State`, `@StateObject`/`@Observable`, or the environment. Coming from Python where objects persist, this is deeply unintuitive.

4. **Xcode vs your editor** -- You *must* use Xcode for SwiftUI development (previews, Interface Builder integration, signing). There is no "just use VS Code" escape hatch for a macOS app. Xcode's autocomplete and error reporting are... uneven.

### What Will Feel Familiar

- `async/await` maps reasonably well from Python's `asyncio`
- Pattern matching (`switch`/`case`) is powerful and similar to Python 3.10+ `match`
- String interpolation: `"Hello \(name)"` vs `f"Hello {name}"`
- Package management (SPM) is similar in concept to `uv`/`pip`, just via Xcode UI or `Package.swift`

### Recommended Learning Path

1. **Apple's "Introducing SwiftUI" tutorial** (2-3 hours) -- Builds a real app, teaches fundamentals
2. **Hacking with Swift: "SwiftUI by Example"** (reference, dip in as needed) -- ~600 pages of examples
3. **Swift Concurrency by Example** (Hacking with Swift) -- When you hit async/networking
4. **Swift.org "Value and Reference Types"** article -- Critical mental model shift

---

## 2. Realistic Timeline

### With AI Pair Programming (Claude Code)

| Phase | Duration | What You Build |
|-------|----------|----------------|
| **Week 0.5**: Swift/SwiftUI basics | 2-3 days | Apple tutorial app, experiment with `@State`, `@Observable`, navigation |
| **Week 1**: PDF + Data model | 3-4 days | PDFKit chapter extraction, GRDB schema, Codable models |
| **Week 1.5**: Claude API integration | 2-3 days | SwiftAnthropic setup, prompt engineering, JSON parsing |
| **Week 2**: Flashcard UI | 3-4 days | Card flip animation, grading buttons, deck navigation |
| **Week 2.5**: Polish + Server | 2-3 days | Local HTTP server (Vapor), review scheduling (SM-2), error handling |
| **Week 3**: Integration testing | 2-3 days | End-to-end flow, edge cases, crash fixes |

**Total: 2.5-4 weeks** (assuming 4-6 hours/day, with AI assistance).

### Without AI Pair Programming

Double it. 5-8 weeks. Most of the extra time is spent deciphering Swift compiler errors and understanding SwiftUI's state management model.

### Key Risk: Xcode Friction

Budget extra time for:
- Xcode project configuration (signing, capabilities, entitlements)
- SwiftUI preview crashes (they happen constantly)
- SPM dependency resolution failures
- Debugging with LLDB instead of `print()` debugging (though `print()` works fine for a PoC)

---

## 3. Key Swift Packages

### Required

| Package | Purpose | Maturity | Notes |
|---------|---------|----------|-------|
| **PDFKit** (Apple framework) | PDF loading, text extraction, outline/TOC | Built-in | No package needed. Part of macOS/iOS SDK. |
| **SwiftAnthropic** | Claude API client | Good (232 stars) | Community-maintained. Supports Messages API, streaming, vision, PDF, function calling, extended thinking. SPM install. iOS 15+/macOS 12+. |
| **GRDB.swift** | SQLite database | Excellent (8.3k stars, 11k+ commits) | The gold standard for Swift SQLite. Migrations, Codable records, database observation for SwiftUI, async support. |
| **Vapor** | HTTP server | Excellent (26k stars) | Full-featured web framework. Overkill for a localhost endpoint but well-documented and battle-tested. |

### Alternatives Considered

| Instead of... | Consider... | Trade-off |
|---------------|-------------|-----------|
| SwiftAnthropic | Raw `URLSession` + REST API | More control, but you rewrite JSON parsing, streaming, error handling. Not worth it for a PoC. |
| GRDB.swift | SwiftData (Apple) | SwiftData is newer, Apple-native, but less flexible for raw SQL. GRDB is better if you want SQLite control similar to Python's `sqlite3`. |
| GRDB.swift | SQLite.swift | GRDB is more actively maintained and has better SwiftUI integration. |
| Vapor | Hummingbird | Lighter weight, but less documentation. Vapor has better learning resources. |
| Vapor | `swift-nio` raw | Way too low-level for a PoC. Use Vapor. |
| Vapor | Embassy / Swifter | Simpler HTTP servers, but less maintained. |

### Package.swift Dependencies

```swift
// Package.swift (or via Xcode Add Package Dependency)
dependencies: [
    .package(url: "https://github.com/jamesrochabrun/SwiftAnthropic", from: "1.0.0"),
    .package(url: "https://github.com/groue/GRDB.swift", from: "7.0.0"),
    .package(url: "https://github.com/vapor/vapor", from: "4.0.0"),
]
```

---

## 4. PDFKit Chapter Extraction: How Well Does It Work?

### The Good

- **PDFKit is a mature Apple framework** (ships with macOS since 10.4, iOS since 11.0). No third-party dependency needed.
- `PDFDocument` provides `outlineRoot` which gives access to the PDF's Table of Contents / bookmarks as a tree of `PDFOutline` objects.
- Each `PDFOutline` node has a `label` (chapter title), `destination` (page reference), and `numberOfChildren` for sub-chapters.
- `PDFPage.string` extracts the full text content of a page.

### The Gotchas (Critical -- Based on Your PyMuPDF Experience)

You already know these from `notebooklm-pdf-by-chapters`, but they apply equally in Swift:

1. **Not all PDFs have TOC bookmarks.** Many ebooks and scanned PDFs have zero outline entries. `outlineRoot` will be `nil`. You need a fallback strategy (regex-based chapter detection on text, or user-specified page ranges).

2. **TOC accuracy varies wildly.** Publisher-generated TOCs are usually accurate. Author-generated ones may point to wrong pages, have inconsistent nesting levels, or use non-standard labeling.

3. **Text extraction quality depends on the PDF.** Native-text PDFs work well. Scanned/OCR PDFs produce garbage or nothing. PDFKit does NOT include OCR -- you would need Apple's Vision framework (`VNRecognizeTextRequest`) for scanned pages.

4. **Page ranges between chapters.** PDFKit outlines give you the *start* page of each chapter. You must calculate the end page as "next chapter start - 1" (same logic as your Python splitter).

5. **Multi-column layouts and figures.** `PDFPage.string` extracts text in reading order, which may interleave columns or include figure captions in unexpected positions.

### PDFKit Chapter Extraction Pattern (Swift)

```swift
import PDFKit

func extractChapters(from url: URL) -> [(title: String, text: String)] {
    guard let document = PDFDocument(url: url),
          let outline = document.outlineRoot else {
        return [] // No TOC available
    }

    var chapters: [(title: String, text: String)] = []

    for i in 0..<outline.numberOfChildren {
        guard let child = outline.child(at: i),
              let title = child.label,
              let destination = child.destination,
              let startPage = destination.page else { continue }

        let startIndex = document.index(for: startPage)

        // End page = next chapter start or end of document
        let endIndex: Int
        if i + 1 < outline.numberOfChildren,
           let nextChild = outline.child(at: i + 1),
           let nextDest = nextChild.destination,
           let nextPage = nextDest.page {
            endIndex = document.index(for: nextPage) - 1
        } else {
            endIndex = document.pageCount - 1
        }

        // Extract text from page range
        var text = ""
        for pageIndex in startIndex...endIndex {
            if let page = document.page(at: pageIndex) {
                text += page.string ?? ""
                text += "\n"
            }
        }

        chapters.append((title: title, text: text))
    }

    return chapters
}
```

### Comparison: PDFKit vs PyMuPDF (Your Existing Tool)

| Feature | PDFKit (Swift) | PyMuPDF (Python) |
|---------|---------------|-----------------|
| TOC extraction | `outlineRoot` tree | `doc.get_toc()` list |
| Text extraction | `page.string` | `page.get_text()` |
| OCR support | No (need Vision framework) | Yes (via Tesseract integration) |
| PDF splitting/saving | Not built-in (need CGContext) | `doc.select()` + `save()` |
| Cross-platform | macOS/iOS only | Everywhere |
| Performance | Good | Excellent |

**Bottom line**: PDFKit handles TOC extraction and text extraction well for well-formed PDFs. It is roughly equivalent to PyMuPDF for your use case, minus the ability to save split chapter PDFs (which you do not need for the flashcard app -- you just need the text).

---

## 5. Calling Claude API from Swift

### No Official Anthropic Swift SDK

As of March 2026, Anthropic provides official SDKs for: **Python, TypeScript, Java, Go, Ruby, C#, PHP**. There is no official Swift SDK.

### Best Option: SwiftAnthropic (Community Package)

- **Repository**: https://github.com/jamesrochabrun/SwiftAnthropic
- **Stars**: 232 (active development, 137 commits)
- **Platform support**: iOS 15+, macOS 12+, Linux (via AsyncHTTPClient)
- **License**: MIT

**Supported Anthropic endpoints:**
- Messages API (create, stream)
- Function calling (tool use)
- Vision (image input)
- PDF support (direct PDF input)
- Prompt caching
- Extended thinking
- Token counting
- Web search tool
- Text editor tool

**Usage example (Messages API):**

```swift
import SwiftAnthropic

let service = AnthropicServiceFactory.service(apiKey: "your-api-key")

let message = MessageParameter(
    model: .claude4Sonnet,  // Check for latest model constants
    messages: [
        .init(role: .user, content: .text("Extract flashcards from this text as JSON..."))
    ],
    maxTokens: 4096
)

let response = try await service.createMessage(message)
```

**Streaming:**

```swift
let stream = try await service.streamMessage(message)
for try await event in stream {
    // Handle streaming events
}
```

### Alternative: Raw URLSession

If SwiftAnthropic does not cover your needs, the Anthropic REST API is straightforward:

```swift
var request = URLRequest(url: URL(string: "https://api.anthropic.com/v1/messages")!)
request.httpMethod = "POST"
request.setValue("application/json", forHTTPHeaderField: "content-type")
request.setValue("your-api-key", forHTTPHeaderField: "x-api-key")
request.setValue("2023-06-01", forHTTPHeaderField: "anthropic-version")

let body: [String: Any] = [
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 4096,
    "messages": [["role": "user", "content": "..."]]
]
request.httpBody = try JSONSerialization.data(withJSONObject: body)

let (data, _) = try await URLSession.shared.data(for: request)
let response = try JSONDecoder().decode(AnthropicResponse.self, from: data)
```

**Recommendation**: Use SwiftAnthropic for the PoC. It handles streaming, error types, and model constants. Only fall back to raw URLSession if you hit a blocking issue.

### API Key Security Note

SwiftAnthropic's README warns: "Your API key is a secret! Do not expose it in client-side code." For a local-only macOS PoC, storing the key in an environment variable or macOS Keychain is fine. For distribution, you would need a backend proxy.

---

## 6. Recommended Project Structure

### Xcode Multiplatform App (macOS + iOS Shared)

Xcode's "Multiplatform" app template creates the right structure automatically:

```
FlashcardApp/
  FlashcardApp.xcodeproj

  Shared/                          # All shared SwiftUI code
    FlashcardApp.swift             # @main App entry point
    ContentView.swift              # Root navigation

    Models/
      Flashcard.swift              # Codable + GRDB Record
      Deck.swift                   # Collection of flashcards
      ReviewSession.swift          # SM-2 review data
      Chapter.swift                # Extracted PDF chapter

    Views/
      DeckListView.swift           # List of flashcard decks
      FlashcardView.swift          # Single card with flip animation
      ReviewView.swift             # Card review session with grading
      PDFImportView.swift          # File picker + chapter selection
      ProgressView.swift           # Review statistics

    ViewModels/                    # @Observable classes
      DeckViewModel.swift          # Business logic for deck operations
      ReviewViewModel.swift        # SM-2 scheduling logic
      ImportViewModel.swift        # PDF extraction + Claude API orchestration

    Services/
      ClaudeService.swift          # SwiftAnthropic wrapper
      PDFExtractor.swift           # PDFKit chapter extraction
      DatabaseService.swift        # GRDB setup, migrations, queries
      ReviewScheduler.swift        # SM-2 algorithm implementation

    Database/
      AppDatabase.swift            # GRDB DatabaseQueue setup
      Migrations.swift             # Schema migrations

  macOS/                           # macOS-specific
    macOSApp.swift                 # Platform-specific entry (if needed)
    MenuBarView.swift              # macOS menu bar extras

  iOS/                             # iOS-specific
    iOSApp.swift                   # Platform-specific entry (if needed)

  Server/                          # Vapor localhost server
    ServerMain.swift               # Vapor app setup
    Routes.swift                   # API endpoints for MCP

  Resources/
    Assets.xcassets                # App icons, colors

  Tests/
    FlashcardAppTests/
      PDFExtractorTests.swift
      ReviewSchedulerTests.swift
      DatabaseTests.swift
```

### Key Architectural Decisions

1. **MVVM pattern** -- Standard for SwiftUI apps. Views observe ViewModels, ViewModels call Services.

2. **`@Observable` macro (iOS 17+ / macOS 14+)** -- The modern way to make objects observable in SwiftUI. Replaces the older `ObservableObject` + `@Published` pattern. Since this is a new app, use the new API.

3. **GRDB over SwiftData** -- SwiftData is Apple's newer ORM, but GRDB gives you direct SQLite access similar to Python's `sqlite3` module. Better for someone who already thinks in SQL.

4. **Vapor as a separate target** -- The localhost server should be a separate executable target in your Xcode project (or a separate Swift package). Do not embed Vapor inside the SwiftUI app process for production, though for a PoC it is fine to run it in-process on a background thread.

5. **Platform branching with `#if os(macOS)`** -- Use conditional compilation for platform-specific features (e.g., `NSOpenPanel` for file picking on macOS vs `.fileImporter()` on iOS).

---

## 7. Swift-Specific Gotchas for a Python Developer

### Memory Management

- **ARC (Automatic Reference Counting)**, not garbage collection. Similar to Python's reference counting but without a cycle collector.
- **Retain cycles are YOUR problem.** If object A holds a strong reference to B, and B holds a strong reference to A, neither gets deallocated. Use `[weak self]` in closures.
- **For a PoC, you probably will not hit this.** Retain cycles matter in long-running apps with complex object graphs. A flashcard app with a few view models is unlikely to leak significantly.

```swift
// WRONG: potential retain cycle
service.onComplete = {
    self.updateUI()  // self captured strongly
}

// RIGHT: weak capture
service.onComplete = { [weak self] in
    self?.updateUI()  // self captured weakly, nil-checked
}
```

### async/await Differences from Python

| Python asyncio | Swift Concurrency |
|----------------|-------------------|
| `async def` / `await` | `func x() async` / `await x()` |
| Event loop (explicit with `asyncio.run()`) | Runtime-managed (no explicit loop) |
| `asyncio.gather()` | `async let` or `TaskGroup` |
| `asyncio.Task()` | `Task { }` |
| No thread safety guarantees | `@Sendable`, `actor` isolation |
| GIL means single-threaded | True parallelism |

**The big difference**: Swift's concurrency is *thread-safe by default*. The compiler enforces `Sendable` conformance -- you cannot pass non-thread-safe types across concurrency boundaries. This causes confusing compiler errors early on.

**`@MainActor`**: SwiftUI views run on the main thread. Any `@Observable` class that updates the UI must be `@MainActor`. This is like Python's "update the UI from the main thread" rule, but enforced at compile time.

```swift
@MainActor
@Observable
class DeckViewModel {
    var flashcards: [Flashcard] = []

    func loadCards() async {
        // This runs on the main actor
        let cards = await DatabaseService.shared.fetchCards()  // async call
        self.flashcards = cards  // Safe: we're on @MainActor
    }
}
```

### Type System Shock

- **No `Any` escape hatch** (well, there is `Any`, but using it is a code smell and loses all type safety).
- **Generics are pervasive.** SwiftUI's `View` protocol uses associated types (`Body`). Error messages about generic constraints are notoriously unhelpful.
- **`Codable` is your friend.** Swift's `Codable` protocol (= `Encodable & Decodable`) auto-generates JSON serialization for structs. Much less boilerplate than Python's `dataclasses` + manual JSON handling.

```swift
// This single declaration gives you JSON encode/decode for free
struct Flashcard: Codable, Identifiable {
    let id: UUID
    var question: String
    var answer: String
    var difficulty: Int
    var nextReviewDate: Date
}

// Decode from JSON
let flashcard = try JSONDecoder().decode(Flashcard.self, from: jsonData)

// Encode to JSON
let data = try JSONEncoder().encode(flashcard)
```

### Xcode-Specific Pain

1. **Build times.** Swift compilation is slower than Python interpretation. A clean build of a medium project takes 30-60 seconds. Incremental builds are faster but still noticeable.

2. **Preview crashes.** SwiftUI previews are powerful but fragile. They crash on runtime errors, async code issues, and sometimes for no apparent reason. When previews break, use the simulator instead.

3. **Code signing.** Even for local development, Xcode requires a signing identity. Free Apple ID works, but you will see warnings and capability limitations.

4. **No REPL-driven development.** Python's `python -i` or Jupyter workflow does not exist. Swift has a REPL (`swift` command) and Playgrounds, but they do not support SwiftUI or framework imports well. Your feedback loop is: edit -> build -> run/preview.

### String Handling

- Swift strings are Unicode-correct by default (like Python 3), but indexing is different.
- No `string[5]` integer indexing. Must use `String.Index` types.
- String interpolation: `"Card \(index) of \(total)"` -- similar to f-strings.

### Error Handling

```swift
// Swift                          // Python equivalent
do {                              // try:
    let data = try loadPDF()      //     data = load_pdf()
    let cards = try parse(data)   //     cards = parse(data)
} catch PDFError.noOutline {      // except PDFError as e:
    print("No TOC found")         //     if isinstance(e, NoOutline):
} catch {                         // except Exception as e:
    print("Error: \(error)")      //     print(f"Error: {e}")
}
```

The `try` keyword must appear before every throwing call (unlike Python where one `try` block covers everything). This is verbose but makes error sources explicit.

---

## 8. The Elephant in the Room: Should You Even Do This in Swift?

### Arguments FOR Swift/SwiftUI

- Native macOS feel (menus, keyboard shortcuts, drag-and-drop, system PDF handling)
- PDFKit is a first-class citizen on macOS -- no third-party dependency for PDF handling
- SwiftUI animations (card flip) are trivial compared to web CSS/JS
- Learning Swift adds a valuable skill for potential iOS/macOS app distribution
- Local-first architecture with GRDB is fast and private
- Future: distribute via Mac App Store or TestFlight

### Arguments AGAINST (Extend Python Stack Instead)

- **You already have PDF chapter extraction in Python** (`notebooklm-pdf-by-chapters` with PyMuPDF)
- **You already have a working study system** (Socratic Study Mentor with Textual TUI + PWA)
- **The web PWA already has flashcards, Pomodoro, voice** -- adding card flip animation to the PWA is a few hours of CSS
- **Python + Claude API is a known quantity** -- official SDK, well-tested
- **Learning Swift is a significant investment** for what might be a PoC that stays a PoC
- **Cross-platform**: The Python/web version works everywhere. A Swift app is Apple-only
- **MCP integration is Python-native** -- the MCP SDK is Python/TypeScript, not Swift

### The Hybrid Option (Best of Both Worlds)

Keep the Python backend (PDF extraction, Claude API, SQLite, MCP server) and build a **thin SwiftUI frontend** that talks to it via localhost HTTP:

```
Python Backend (existing code)          SwiftUI Frontend (new)
  - PDF extraction (PyMuPDF)              - Native macOS UI
  - Claude API (official SDK)             - Card flip animations
  - SQLite (existing schema)              - Grading interface
  - FastAPI/MCP server                    - Reads from localhost API
```

This approach:
- Reuses all your existing Python code
- Gives you native macOS UI where it matters (the flashcard review experience)
- Limits Swift learning to just SwiftUI views + URLSession networking
- Timeline: ~1-1.5 weeks instead of 3-4

---

## 9. Decision Framework

| If your goal is... | Then... |
|---------------------|---------|
| Learn Swift properly | Build the full Swift PoC. Accept the 3-4 week timeline. |
| Ship a better flashcard experience fast | Extend the PWA with better animations, or build the hybrid option. |
| Eventually publish on the App Store | Full Swift is the right investment. Start with the PoC. |
| Explore MCP integration | Stay in Python. MCP tooling is Python/TS-first. |
| Scratch the curiosity itch | Start with Swift Playgrounds + the Apple SwiftUI tutorial. See if you enjoy it before committing to a full PoC. |

---

## 10. Recommended Resources

### Essential (Read/Do These)

1. **Apple "Introducing SwiftUI" tutorial**: https://developer.apple.com/tutorials/swiftui
2. **Hacking with Swift "SwiftUI by Example"**: https://www.hackingwithswift.com/quick-start/swiftui (free, ~600 pages)
3. **Swift.org "Value and Reference Types"**: https://www.swift.org/documentation/articles/value-and-reference-types.html
4. **Hacking with Swift "Swift Concurrency by Example"**: https://www.hackingwithswift.com/quick-start/concurrency

### Package Documentation

5. **GRDB.swift**: https://github.com/groue/GRDB.swift (8.3k stars, excellent docs with SwiftUI integration guides)
6. **SwiftAnthropic**: https://github.com/jamesrochabrun/SwiftAnthropic (community Claude API client)
7. **Vapor**: https://github.com/vapor/vapor (26k stars, docs at https://docs.vapor.codes)

### Reference When Stuck

8. **Hacking with Swift "Common SwiftUI Errors"**: https://www.hackingwithswift.com/quick-start/swiftui/common-swiftui-errors-and-how-to-fix-them
9. **Apple PDFKit docs**: https://developer.apple.com/documentation/pdfkit

---

## Appendix A: SwiftUI Card Flip Animation (It Really Is This Simple)

```swift
struct FlashcardView: View {
    let question: String
    let answer: String
    @State private var isFlipped = false

    var body: some View {
        ZStack {
            // Front (question)
            CardFace(text: question, color: .blue)
                .opacity(isFlipped ? 0 : 1)
                .rotation3DEffect(
                    .degrees(isFlipped ? 180 : 0),
                    axis: (x: 0, y: 1, z: 0)
                )

            // Back (answer)
            CardFace(text: answer, color: .green)
                .opacity(isFlipped ? 1 : 0)
                .rotation3DEffect(
                    .degrees(isFlipped ? 0 : -180),
                    axis: (x: 0, y: 1, z: 0)
                )
        }
        .onTapGesture {
            withAnimation(.spring(duration: 0.6)) {
                isFlipped.toggle()
            }
        }
    }
}

struct CardFace: View {
    let text: String
    let color: Color

    var body: some View {
        RoundedRectangle(cornerRadius: 16)
            .fill(color.opacity(0.15))
            .overlay(
                Text(text)
                    .font(.title2)
                    .padding()
            )
            .frame(width: 400, height: 300)
            .shadow(radius: 4)
    }
}
```

This is genuinely one of SwiftUI's strengths. The equivalent in HTML/CSS/JS is significantly more code.

---

## Appendix B: GRDB Record Example (Flashcard Model)

```swift
import GRDB

struct Flashcard: Codable, Identifiable, FetchableRecord, PersistableRecord {
    var id: Int64?
    var deckId: Int64
    var question: String
    var answer: String
    var easeFactor: Double = 2.5       // SM-2
    var interval: Int = 0              // Days until next review
    var repetitions: Int = 0
    var nextReviewDate: Date = Date()

    // Auto-generate column names from properties
    static let databaseTableName = "flashcard"

    // Auto-set id after insert
    mutating func didInsert(_ inserted: InsertionSuccess) {
        id = inserted.rowID
    }
}

// Migration
var migrator = DatabaseMigrator()
migrator.registerMigration("v1") { db in
    try db.create(table: "flashcard") { t in
        t.autoIncrementedPrimaryKey("id")
        t.column("deckId", .integer).notNull()
        t.column("question", .text).notNull()
        t.column("answer", .text).notNull()
        t.column("easeFactor", .double).notNull().defaults(to: 2.5)
        t.column("interval", .integer).notNull().defaults(to: 0)
        t.column("repetitions", .integer).notNull().defaults(to: 0)
        t.column("nextReviewDate", .datetime).notNull()
    }
}
```

This maps almost 1:1 to how you define models in Python with dataclasses + sqlite3. GRDB's `Codable` integration means zero manual column mapping.
