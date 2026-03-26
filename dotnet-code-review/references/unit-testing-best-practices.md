# .NET Unit Testing Best Practices

Reference for the `dotnet-code-review` skill. Covers xUnit, NUnit, and MSTest conventions, plus Moq and NSubstitute mocking patterns.

---

## 1. Test Naming

### Unclear test names — WARNING
Test names are documentation. A failing test whose name is `Test1` or `GetStory_Test` tells you nothing about what broke or why. Follow the three-part naming convention.

**Format:** `MethodName_Scenario_ExpectedBehaviour`

```csharp
// Avoid: tells you nothing when it fails
[Fact]
public void Test_GetStory() { ... }

// Prefer: readable as a sentence describing the behaviour
[Fact]
public void GetStory_WhenIdDoesNotExist_ReturnsNull() { ... }

[Fact]
public void PollAndPublishStories_WhenApiReturnsEmpty_LogsWarningAndReturnsEarly() { ... }

[Fact]
public void Add_WhenStoreAtCapacity_EvictsOldestEntry() { ... }
```

> Ref: https://learn.microsoft.com/dotnet/core/testing/unit-testing-best-practices#follow-test-naming-standards

---

## 2. Structure: Arrange / Act / Assert

Every test should have a clear AAA structure. Mixing arrangement and assertion makes tests harder to read and debug.

```csharp
// Avoid: arrangement, action, and assertion are interleaved
[Fact]
public void QueryReturnsFilteredStories()
{
    var store = new StoryStore(10);
    store.Upsert(new HackerNewsStory { Id = 1, Score = 5 });
    Assert.Empty(store.Query(minScore: 10, null, null));
    store.Upsert(new HackerNewsStory { Id = 2, Score = 15 });
    Assert.Single(store.Query(minScore: 10, null, null));
}

// Prefer: clear separation of phases
[Fact]
public void Query_WhenMinScoreFilter_ExcludesStoriesBelowThreshold()
{
    // Arrange
    var store = new StoryStore(10);
    store.Upsert(new HackerNewsStory { Id = 1, Score = 5 });
    store.Upsert(new HackerNewsStory { Id = 2, Score = 15 });

    // Act
    var results = store.Query(minScore: 10, start: null, end: null);

    // Assert
    Assert.Single(results);
    Assert.Equal(2, results[0].Id);
}
```

> Ref: https://learn.microsoft.com/dotnet/core/testing/unit-testing-best-practices#arrange-your-tests

---

## 3. One Concern Per Test

### Testing multiple behaviours in one test — WARNING
A test that asserts many things becomes difficult to diagnose when it fails. One test should verify one logical behaviour. Multiple `Assert` calls on the same result object are fine; testing multiple distinct scenarios is not.

```csharp
// Avoid: two separate concerns in one test — which one failed?
[Fact]
public void StoryStore_Works()
{
    var store = new StoryStore(5);
    store.Upsert(new HackerNewsStory { Id = 1, Score = 10 });
    Assert.True(store.TryGet(1, out _));       // concern 1: upsert works
    Assert.False(store.TryGet(99, out _));     // concern 2: missing id returns false
    // ... 8 more assertions on different behaviours
}

// Prefer: one test per concern
[Fact]
public void TryGet_AfterUpsert_ReturnsTrueAndStory() { ... }

[Fact]
public void TryGet_WhenIdNotPresent_ReturnsFalse() { ... }
```

---

## 4. Avoid Infrastructure in Unit Tests

### Hitting real databases, file systems, or network in unit tests — WARNING
Unit tests must be fast and deterministic. Real infrastructure makes tests slow, order-dependent, and reliant on environment setup. Mock or stub external dependencies; reserve real infrastructure for integration tests.

```csharp
// Avoid: unit test that depends on a real SQS queue
[Fact]
public async Task PublishStory_SendsMessageToSqs()
{
    var client = new AmazonSQSClient(); // real AWS call
    var service = new StoryPollingService(client, ...);
    await service.PollAndPublishStoriesAsync(CancellationToken.None);
    // assertion requires a running queue to verify
}

// Prefer: mock the SQS client
[Fact]
public async Task PublishStory_WhenStoryIsRelevant_SendsOneMessage()
{
    var sqsClient = Substitute.For<IAmazonSQS>(); // NSubstitute
    var service = BuildService(sqsClient);
    await service.PollAndPublishStoriesAsync(CancellationToken.None);
    await sqsClient.Received(1).SendMessageAsync(Arg.Any<SendMessageRequest>(), Arg.Any<CancellationToken>());
}
```

> Ref: https://learn.microsoft.com/dotnet/core/testing/unit-testing-best-practices#avoid-infrastructure-dependencies

---

## 5. NSubstitute Patterns

NSubstitute uses a fluent, natural-language API. The substitute is created with `Substitute.For<T>()`.

### Basic setup and verification

```csharp
var apiService = Substitute.For<IHackerNewsApiService>();

// Setup: return a value when called
apiService.GetTopStoryIdsAsync(Arg.Any<CancellationToken>())
    .Returns(new[] { 1, 2, 3 });

// Setup: return null to simulate empty response
apiService.GetStoryAsync(Arg.Any<int>(), Arg.Any<CancellationToken>())
    .Returns((HackerNewsStory?)null);

// Verify: method was called exactly once with any args
await apiService.Received(1).GetTopStoryIdsAsync(Arg.Any<CancellationToken>());

// Verify: method was never called
await apiService.DidNotReceive().GetStoryAsync(Arg.Any<int>(), Arg.Any<CancellationToken>());
```

### Async methods

NSubstitute handles `Task`-returning methods naturally — `.Returns(value)` is automatically wrapped in a completed Task.

```csharp
// No need to wrap in Task.FromResult — NSubstitute does it automatically
apiService.GetTopStoryIdsAsync(Arg.Any<CancellationToken>())
    .Returns(new[] { 1, 2, 3 });

// For Task<T> where you want to return null explicitly
apiService.GetStoryAsync(Arg.Any<int>(), Arg.Any<CancellationToken>())
    .Returns(Task.FromResult<HackerNewsStory?>(null));
```

### Throwing exceptions

```csharp
// Throw on call
apiService.GetTopStoryIdsAsync(Arg.Any<CancellationToken>())
    .Throws(new HttpRequestException("timeout"));

// Throw async
apiService.GetTopStoryIdsAsync(Arg.Any<CancellationToken>())
    .ThrowsAsync(new HttpRequestException("timeout"));
```

### Argument matching

```csharp
// Match any value
Arg.Any<int>()

// Match a specific value
Arg.Is<int>(id => id > 0)

// Exact value (positional)
await sqsClient.Received().SendMessageAsync(
    Arg.Is<SendMessageRequest>(r => r.QueueUrl == "https://my-queue"),
    Arg.Any<CancellationToken>());
```

---

## 6. Moq Patterns

Moq uses a `Mock<T>` wrapper. The underlying object is accessed via `.Object`.

### Basic setup and verification

```csharp
var mockApiService = new Mock<IHackerNewsApiService>();

// Setup: return a value
mockApiService
    .Setup(x => x.GetTopStoryIdsAsync(It.IsAny<CancellationToken>()))
    .ReturnsAsync(new[] { 1, 2, 3 });

// Setup: return null
mockApiService
    .Setup(x => x.GetStoryAsync(It.IsAny<int>(), It.IsAny<CancellationToken>()))
    .ReturnsAsync((HackerNewsStory?)null);

// Pass the mock to the system under test
var service = new StoryPollingService(mockApiService.Object, ...);

// Verify: called exactly once
mockApiService.Verify(
    x => x.GetTopStoryIdsAsync(It.IsAny<CancellationToken>()),
    Times.Once);

// Verify: never called
mockApiService.Verify(
    x => x.GetStoryAsync(It.IsAny<int>(), It.IsAny<CancellationToken>()),
    Times.Never);
```

### Async methods — use `ReturnsAsync`, not `Returns(Task.FromResult(...))`

```csharp
// Avoid: verbose and easy to get wrong
mockApiService
    .Setup(x => x.GetTopStoryIdsAsync(It.IsAny<CancellationToken>()))
    .Returns(Task.FromResult(new[] { 1, 2, 3 }));

// Prefer: cleaner with ReturnsAsync
mockApiService
    .Setup(x => x.GetTopStoryIdsAsync(It.IsAny<CancellationToken>()))
    .ReturnsAsync(new[] { 1, 2, 3 });
```

### Throwing exceptions

```csharp
// Sync throw
mockApiService
    .Setup(x => x.GetTopStoryIdsAsync(It.IsAny<CancellationToken>()))
    .ThrowsAsync(new HttpRequestException("timeout"));
```

### Argument matching

```csharp
It.IsAny<int>()                    // any value of type
It.Is<int>(id => id > 0)           // conditional match
It.IsIn(1, 2, 3)                   // value in set
It.IsRegex("^https://")            // string regex match
```

### `MockBehavior.Strict` — catch unexpected calls

```csharp
// Avoid: loose mocks silently return default values for unsetup calls,
// which can mask bugs where a method is called unexpectedly
var mock = new Mock<IHackerNewsApiService>();

// Prefer: strict mode throws if any non-setup method is called
var mock = new Mock<IHackerNewsApiService>(MockBehavior.Strict);
```

---

## 7. Common Mocking Anti-Patterns

### Mocking concrete classes or types you own — WARNING
Mock at system boundaries (external services, I/O). Mocking your own concrete classes couples tests to implementation details and is usually a sign the class should be extracted behind an interface.

```csharp
// Avoid: mocking a concrete implementation you own
var mock = new Mock<StoryStore>(); // StoryStore is your own class

// Prefer: use the real StoryStore in unit tests (it's an in-memory object)
var store = new StoryStore(maxCount: 10);
```

---

### Not resetting or isolating mocks between tests — WARNING
Shared mock instances across tests can carry state from one test into another. Create a fresh substitute/mock per test, or use a test fixture with proper setup.

```csharp
// Avoid: static or shared mock used by multiple tests
private static readonly Mock<IAmazonSQS> _sharedMock = new();

// Prefer: create per test (xUnit creates a new class instance per test by default)
private readonly Mock<IAmazonSQS> _sqsMock = new();
```

---

### Verifying internal implementation steps instead of observable outcomes — WARNING
Tests should verify what the unit produces (return values, state changes, calls to dependencies), not how it does it internally. Over-specifying internal behaviour makes tests brittle.

```csharp
// Avoid: verifying every internal step
mockApiService.Verify(x => x.GetTopStoryIdsAsync(...), Times.Once);
mockApiService.Verify(x => x.GetStoryAsync(1, ...), Times.Once);
mockApiService.Verify(x => x.GetStoryAsync(2, ...), Times.Once);
// ... 10 more verifies on internal ordering

// Prefer: verify the observable outcome — the message was published
await sqsClient.Received(1).SendMessageAsync(
    Arg.Is<SendMessageRequest>(r => r.MessageBody.Contains("\"Id\":1")),
    Arg.Any<CancellationToken>());
```

---

## 8. Testing Async Code

### Not awaiting async assertions — CRITICAL
Forgetting to `await` an async test method means the test completes before the assertion runs, producing a false pass.

```csharp
// Avoid: test exits before the assertion is evaluated
[Fact]
public void PollAndPublish_PublishesMatchingStory()
{
    // missing async/await — service.PollAndPublishStoriesAsync returns a Task that is never awaited
    _service.PollAndPublishStoriesAsync(CancellationToken.None);
    _sqsMock.Verify(x => x.SendMessageAsync(...), Times.Once); // always passes
}

// Prefer: async test method with await
[Fact]
public async Task PollAndPublish_WhenStoryIsRelevant_PublishesMatchingStory()
{
    await _service.PollAndPublishStoriesAsync(CancellationToken.None);
    await _sqsSubstitute.Received(1).SendMessageAsync(Arg.Any<SendMessageRequest>(), Arg.Any<CancellationToken>());
}
```

---

### Testing that an exception is thrown

```csharp
// xUnit
await Assert.ThrowsAsync<InvalidOperationException>(
    () => _service.DoWorkAsync(CancellationToken.None));

// NSubstitute setup
_apiService.GetTopStoryIdsAsync(Arg.Any<CancellationToken>())
    .ThrowsAsync(new HttpRequestException("network error"));

// Moq setup
_mockApiService
    .Setup(x => x.GetTopStoryIdsAsync(It.IsAny<CancellationToken>()))
    .ThrowsAsync(new HttpRequestException("network error"));
```

---

## 9. Test Organisation

### Using `[SetUp]` / `[TearDown]` for shared state — SUGGESTION
These attributes exist in NUnit and MSTest, but xUnit removed them intentionally. Prefer constructor-based setup and `IDisposable` teardown. In all frameworks, shared setup that hides preconditions makes tests harder to reason about.

```csharp
// Avoid: hidden setup creates unclear test preconditions
[SetUp]
public void Setup() { _store = new StoryStore(10); }

// Prefer (xUnit): constructor is the setup; IDisposable is the teardown
public class StoryStoreTests : IDisposable
{
    private readonly StoryStore _store = new(10);

    [Fact]
    public void Query_ReturnsStoriesNewestFirst() { ... }

    public void Dispose() { /* cleanup if needed */ }
}
```

> Ref: https://learn.microsoft.com/dotnet/core/testing/unit-testing-best-practices#use-helper-methods-instead-of-setup-and-teardown

---

### Use `[Theory]` / `[InlineData]` for data-driven cases — SUGGESTION
When the same logic needs to be verified for multiple inputs, a parameterised test is cleaner than duplicated test methods.

```csharp
// Avoid: duplicated test bodies for each input
[Fact]
public void IsProgrammingRelated_MatchesPython() => Assert.True(Filter("Python tutorial"));
[Fact]
public void IsProgrammingRelated_MatchesDocker() => Assert.True(Filter("Docker guide"));

// Prefer: one theory with multiple data points
[Theory]
[InlineData("Python tutorial")]
[InlineData("Docker guide")]
[InlineData("Learn Kubernetes")]
public void IsProgrammingRelated_RecognisesKeyword_ReturnsTrue(string title)
{
    Assert.True(_filter.IsProgrammingRelated(title));
}
```

---

### Test project should not reference production infrastructure packages — WARNING
If a unit test project references `AWSSDK.SQS`, `Microsoft.EntityFrameworkCore`, or similar infrastructure packages directly, it is a signal that tests are hitting real infrastructure rather than using mocks. Unit test projects should only reference the production project and mocking libraries.
