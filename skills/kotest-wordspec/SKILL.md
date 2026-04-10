---
name: kotest-wordspec
description: Write unit tests using Kotest WordSpec style with MockK, Turbine, and coroutines-test. Use this skill whenever the user asks to create tests, write tests, add test coverage, generate unit tests, or test any Kotlin class — including use cases, ViewModels, repositories, data sources, preferences, services, or mappers. Also use it when the user mentions Kotest, WordSpec, MockK, Turbine, or asks to test Flows, coroutines, or suspend functions in a Kotlin project.
---

# Kotest WordSpec Test Writing

This skill provides instructions for writing unit tests using Kotest WordSpec style in Kotlin projects. It covers the complete testing stack: Kotest WordSpec for structure, MockK for mocking, Turbine for Flow testing, and kotlinx-coroutines-test for coroutine testing.

## Why Kotest WordSpec

WordSpec produces human-readable, BDD-style test structures. Tests read like specifications: `"ClassName" When { "scenario" should { "expected behavior" { ... } } }`. This makes tests self-documenting and easy to navigate. The framework also has first-class coroutine support, so suspend functions work natively inside test blocks without extra wrappers.

## Test File Structure

Every test class extends `WordSpec` using a constructor lambda. The overall structure follows this pattern:

```kotlin
package com.example.feature.domain

import io.kotest.core.spec.style.WordSpec
import io.kotest.matchers.shouldBe
import io.mockk.mockk

class GetUserProfileUseCaseTest :
    WordSpec({

        // 1. Declare dependencies as lateinit var
        lateinit var mockRepository: UserRepository
        lateinit var useCase: GetUserProfileUseCase

        // 2. Initialize in beforeTest (runs before EACH test)
        beforeTest {
            mockRepository = mockk(relaxed = true)
            useCase = GetUserProfileUseCase(repository = mockRepository)
        }

        // 3. Write specs using When/should nesting
        "GetUserProfileUseCase" When {

            "fetching a user profile" should {

                "return the user when found" {
                    // Given
                    // When
                    // Then
                }
            }
        }
    })
```

Key structural rules:
- The class declaration uses `:` followed by a newline, then `WordSpec({` indented
- Use `lateinit var` for all mocked dependencies and the system under test
- Always use `beforeTest { }` (not `@BeforeEach`) to reinitialize mocks before each test — this ensures test isolation
- Close the class with `})` on its own line

## WordSpec Nesting Patterns

WordSpec supports two nesting styles. Choose based on how many distinct scenarios the class under test has.

### Two-level nesting: `When` / `should`

Use this when the class has multiple logical groups of behavior (e.g., different methods, different input states, success vs error paths):

```kotlin
"ClassName" When {

    "scenario or method group" should {

        "specific expected behavior" {
            // test body
        }

        "another expected behavior" {
            // test body
        }
    }

    "different scenario" should {

        "its expected behavior" {
            // test body
        }
    }
}
```

The `When` block groups related scenarios. The `should` block groups assertions for that scenario. The innermost string is the individual test case.

### Single-level nesting: `should` only

Use this when the class has a single logical concern and grouping by `When` would be redundant:

```kotlin
"The SearchItemsUseCase" should {

    "return all items when query is blank" {
        // test body
    }

    "return empty list when no items match" {
        // test body
    }
}
```

### Naming conventions

- `When` block strings: Use the class name or describe the context/scenario. Examples: `"SaveDocumentUseCase"`, `"processing valid input"`, `"handling errors"`, `"ProfileViewModel initialization"`
- `should` block strings: Describe the condition or grouping. Examples: `"the user exists in the database"`, `"handling network failures"`, `"input is valid"`
- Innermost test strings: Describe the expected behavior in plain English. Examples: `"return the saved entity with a generated ID"`, `"emit the default configuration"`, `"return failure when the repository throws"`

## Test Body Pattern: Given / When / Then

Every test body follows the Given/When/Then pattern using comments as section markers:

```kotlin
"return the saved entity with updated timestamp" {
    // Given - Set up test data and mock behavior
    val user = createTestUser(name = "Alice")
    coEvery { repository.save(any()) } returns Result.success(Unit)

    // When - Execute the action under test
    val result = useCase(user)

    // Then - Assert the outcome
    result.isSuccess shouldBe true
    coVerify { repository.save(any()) }
}
```

For short tests where Given and When naturally flow together, you can combine "When & Then":

```kotlin
"emit the default theme" {
    // Given
    val mockPrefs = mockk<Preferences>(relaxed = true)
    every { mockPrefs[themeKey] } returns null

    // When & Then
    settings.themeFlow.test {
        awaitItem() shouldBe Theme.LIGHT
        cancelAndIgnoreRemainingEvents()
    }
}
```

## MockK Usage

### Creating mocks

Always use `mockk(relaxed = true)` unless you need strict verification that unmocked calls should fail:

```kotlin
lateinit var mockRepository: UserRepository

beforeTest {
    mockRepository = mockk(relaxed = true)
}
```

### Stubbing behavior

Use `every` for regular functions and `coEvery` for suspend functions:

```kotlin
// Regular function
every { mockSettings.themeFlow } returns flowOf(Theme.DARK)
every { mockConfig[featureFlagKey] } returns true

// Suspend function
coEvery { repository.save(any()) } returns Result.success(Unit)
coEvery { repository.findById(any()) } returns Result.failure(Exception("Not found"))
```

### Setting up default stubs in beforeTest

When most tests need a "happy path" default, set it up in `beforeTest` and override in specific error tests:

```kotlin
beforeTest {
    mockRepository = mockk(relaxed = true)
    useCase = SaveDocumentUseCase(repository = mockRepository)

    // Default: operations succeed
    coEvery { mockRepository.save(any()) } returns Result.success(Unit)
}
```

### Verification

Use `verify` for regular functions and `coVerify` for suspend functions:

```kotlin
// Simple verification
coVerify { repository.save(any()) }

// Verification with argument matching
coVerify {
    repository.save(
        match { entity ->
            entity.status == Status.ACTIVE &&
                entity.updatedAt != null
        },
    )
}
```

### Capturing arguments with slots

For callback-based APIs or when you need to trigger listener callbacks manually:

```kotlin
val callbackSlot = slot<DataCallback>()

every {
    mockClient.subscribe(capture(callbackSlot))
} returns mockSubscription

// Later, trigger the callback manually:
callbackSlot.captured.onDataReceived(mockData)
```

## Kotest Assertions

Use Kotest matchers (not JUnit assertions or Truth). Common matchers:

```kotlin
import io.kotest.matchers.shouldBe
import io.kotest.matchers.shouldNotBe
import io.kotest.matchers.collections.shouldBeEmpty
import io.kotest.matchers.collections.shouldHaveSize
import io.kotest.matchers.types.shouldBeInstanceOf

// Equality
result shouldBe expected
result shouldNotBe null

// Collections
result.shouldBeEmpty()
result shouldHaveSize 3
result.map { it.id } shouldBe listOf("A", "B", "C")

// Type checking (for sealed classes)
state.shouldBeInstanceOf<UiState.Success>()

// Boolean
result.isSuccess shouldBe true
result.isFailure shouldBe true

// Null checks
result.exceptionOrNull() shouldBe expectedError
entity?.updatedAt shouldNotBe null
```

## Flow Testing with Turbine

Use Turbine's `.test { }` extension to test Flow emissions:

```kotlin
import app.cash.turbine.test

// Basic Flow test
settings.themeFlow.test {
    awaitItem() shouldBe Theme.LIGHT
    cancelAndIgnoreRemainingEvents()
}

// Testing multiple emissions
flow.test {
    awaitItem() shouldBe firstValue
    awaitItem() shouldBe secondValue
    cancelAndIgnoreRemainingEvents()
}

// Testing Flow completion
flow.test {
    awaitItem() shouldBe defaultValue
    awaitComplete()
}

// Testing Flow errors
flow.test {
    awaitError()
}
```

Always call `cancelAndIgnoreRemainingEvents()` when you are done asserting emissions and don't need to verify completion. Use `awaitComplete()` when you specifically want to assert the Flow completes (e.g., after error recovery with a default value).

## ViewModel Testing

ViewModel tests need special coroutine handling because ViewModels use `viewModelScope` which defaults to `Dispatchers.Main`:

```kotlin
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.StandardTestDispatcher
import kotlinx.coroutines.test.advanceUntilIdle
import kotlinx.coroutines.test.resetMain
import kotlinx.coroutines.test.runTest
import kotlinx.coroutines.test.setMain

@OptIn(ExperimentalCoroutinesApi::class)
class ProfileViewModelTest :
    WordSpec({

        lateinit var mockUserRepository: UserRepository
        val testDispatcher = StandardTestDispatcher()

        beforeTest {
            Dispatchers.setMain(testDispatcher)
            mockUserRepository = mockk(relaxed = true)
        }

        afterTest {
            Dispatchers.resetMain()
        }

        "ProfileViewModel initialization" When {

            "the user is logged in" should {

                "emit Success state with user data" {
                    runTest {
                        // Given
                        every { mockUserRepository.getUserFlow() } returns flowOf(testUser)

                        // When
                        val viewModel = ProfileViewModel(userRepository = mockUserRepository)
                        advanceUntilIdle()

                        // Then
                        viewModel.uiState.test {
                            val state = awaitItem()
                            state.shouldBeInstanceOf<ProfileUiState.Success>()
                            (state as ProfileUiState.Success).userName shouldBe "Alice"
                            cancelAndIgnoreRemainingEvents()
                        }
                    }
                }
            }
        }

        "handleEvent" When {

            "a save event is received" should {

                "persist changes and update state" {
                    runTest {
                        // Given
                        every { mockUserRepository.getUserFlow() } returns flowOf(testUser)
                        coEvery { mockUserRepository.save(any()) } returns Result.success(Unit)

                        val viewModel = ProfileViewModel(userRepository = mockUserRepository)
                        advanceUntilIdle()

                        // When
                        viewModel.handleEvent(ProfileEvent.SaveClicked)
                        advanceUntilIdle()

                        // Then
                        coVerify { mockUserRepository.save(any()) }
                    }
                }
            }
        }
    })
```

Key ViewModel testing rules:
- Annotate the class with `@OptIn(ExperimentalCoroutinesApi::class)`
- Create `StandardTestDispatcher()` at spec level (not inside `beforeTest`)
- Set `Dispatchers.setMain(testDispatcher)` in `beforeTest` and `Dispatchers.resetMain()` in `afterTest`
- Wrap each test body in `runTest { }`
- Call `advanceUntilIdle()` after creating the ViewModel or triggering events to let coroutines complete
- Test `uiState` and navigation events with Turbine

## Testing Non-ViewModel Classes

For use cases, repositories, data sources, services, and preferences, Kotest's native coroutine support handles suspend functions automatically. No `runTest` wrapper is needed:

```kotlin
class SaveDocumentUseCaseTest :
    WordSpec({

        lateinit var mockRepo: DocumentRepository
        lateinit var useCase: SaveDocumentUseCase

        beforeTest {
            mockRepo = mockk(relaxed = true)
            useCase = SaveDocumentUseCase(repository = mockRepo)
        }

        "SaveDocumentUseCase" When {

            "saving a valid document" should {

                "return success with the saved document" {
                    // Given
                    val document = createTestDocument(title = "My Report")
                    coEvery { mockRepo.save(any()) } returns Result.success(document)

                    // When
                    val result = useCase(document)

                    // Then
                    result.isSuccess shouldBe true
                    result.getOrThrow().title shouldBe "My Report"
                }
            }

            "handling errors" should {

                "return failure when repository throws" {
                    // Given
                    val document = createTestDocument()
                    val error = Exception("Database write failed")
                    coEvery { mockRepo.save(any()) } returns Result.failure(error)

                    // When
                    val result = useCase(document)

                    // Then
                    result.isFailure shouldBe true
                    result.exceptionOrNull() shouldBe error
                }
            }
        }
    })
```

## Helper Functions for Test Data

Create private helper functions at file level (outside the class) to build test objects with sensible defaults:

```kotlin
class SaveDocumentUseCaseTest :
    WordSpec({
        // ... tests that call createTestDocument()
    })

/**
 * Helper function to create test documents with default values
 */
private fun createTestDocument(
    id: String = "doc_001",
    title: String = "Test Document",
    author: String = "Test Author",
    status: DocumentStatus = DocumentStatus.DRAFT,
): Document =
    Document(
        id = id,
        title = title,
        author = author,
        status = status,
        createdAt = LocalDateTime.parse("2024-01-01T00:00:00"),
        // ... other fields with sensible defaults
    )
```

This pattern keeps test bodies focused on the scenario being tested. Give every parameter a default value so each test only specifies the fields relevant to it.

## Error Testing Patterns

### Testing Result.failure

```kotlin
"handling errors" should {

    "return failure when save fails" {
        // Given
        val saveError = Exception("Database write failed")
        coEvery { repository.save(any()) } returns Result.failure(saveError)

        // When
        val result = useCase(entity)

        // Then
        result.isFailure shouldBe true
        result.exceptionOrNull() shouldBe saveError
    }
}
```

### Testing Flow errors

```kotlin
"an error occurs while reading" should {

    "emit default value via catch block" {
        // Given
        every { dataStore.data } returns flow {
            throw RuntimeException("Read error")
        }

        // When & Then
        settings.configFlow.test {
            awaitItem() shouldBe defaultConfig
            awaitComplete()
        }
    }
}
```

## Formatting Rules

- Use **4-space indentation**
- Use **trailing commas** in multi-line parameter lists
- Keep one blank line between test cases within a `should` block
- Keep one blank line after opening `When {` and `should {` blocks
- Named parameters in constructor calls for readability: `MyUseCase(repository = mockRepo)`
- Ktlint-compatible formatting: break long lines, align closing parentheses

## Checklist Before Submitting Tests

1. Each test has a clear Given/When/Then structure with comments
2. All mocks are initialized in `beforeTest` (not at declaration site)
3. Test names describe expected behavior, not implementation details
4. Happy path and error paths are both covered
5. Helper functions use default parameters for clean test setup
6. Turbine tests call `cancelAndIgnoreRemainingEvents()` or `awaitComplete()`
7. ViewModel tests use `runTest`, `StandardTestDispatcher`, `setMain`/`resetMain`, and `advanceUntilIdle()`
8. Non-ViewModel tests do NOT wrap in `runTest` (Kotest handles coroutines natively)
9. Assertions use Kotest matchers (`shouldBe`, `shouldHaveSize`, etc.), not JUnit or Truth
10. Code passes Ktlint formatting (trailing commas, 4-space indent)
