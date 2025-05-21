# RFC: User-facing API for Hedging 

# Config, Rules and Decisions

Regardless whether we are implementing hedging as a separate client or as part of the `RetryClient`, we need to provide a way for users to configure hedging.

## `HedgingDecision`
```java
public final class HedgingDecision {
    public static HedgingDecision continueHedging(HedgingDelayProvider hedgingDelay) { }

    public static HedgingDecision abortHedging() { ... }

    public static HedgingDecision next() { ... }

    @Nullable HedgingDelayProvider hedgingDelayProvider() { ... }

    @Override
    public String toString() { ... }
}
```

## `HedgingDelayProvider`

Similar to `Backoff` for `RetryClient` it will provide the `RetryClient` or `HedgingClient` with the delay to wait before sending the next hedged request.

```java
    @FunctionalInterface
    public interface HedgingDelayProvider {
        static HedgingDelayProvider fixed(long delayMillis) { ... }
        
        long nextHedgingDelayMillis(ClientRequestContext ctx, int attempt);
    }
    
    final class FixedHedingDelayProvider implements HedgingDelayProvider { ... }
```

## `HedgingRule`

```java
@FunctionalInterface
public interface HedgingRule {
    // (1) static HedgingRule failsafe() { ... }
    static HedgingRule failsafe(HedgingDelayProvider hedgingDelayProvider) { ... };

    static HedgingRule onStatusClass(HttpStatusClass statusClass, HedgingDelayProvider hedgingDelayProvider) { ... }
    static HedgingRule onStatusClass(Iterable<HttpStatusClass> statusClasses, HedgingDelayProvider hedgingDelayProvider) { ... }
    static HedgingRule onServerErrorStatus(HedgingDelayProvider hedgingDelayProvider) { ... }
    
    static HedgingRule onStatus(Iterable<HttpStatus> statuses, HedgingDelayProvider hedgingDelayProvider) { ... }
    static HedgingRule onStatus(BiPredicate<? super ClientRequestContext, ? super HttpStatus> statusFilter, HedgingDelayProvider hedgingDelayProvider) { ... }

    static HedgingRule onException(Class<? extends Throwable> exception, HedgingDelayProvider hedgingDelayProvider) { ... }
    static HedgingRule onException(BiPredicate<? super ClientRequestContext, ? super Throwable> exceptionFilter, HedgingDelayProvider hedgingDelayProvider)  { ... }
    static HedgingRule onException(HedgingDelayProvider hedgingDelayProvider) { ... }
    
    static HedgingRule onUnprocessed(HedgingDelayProvider hedgingDelayProvider) { ... }

    // matches HedgingRule.builder(...) methods
    static HedgingRuleBuilder builder() { ... }
    static HedgingRuleBuilder builder(HttpMethod... methods) { ... }
    static HedgingRuleBuilder builder(Iterable<HttpMethod> methods) { ... }
    static HedgingRuleBuilder builder(BiPredicate<? super ClientRequestContext, ? super RequestHeaders> requestHeadersFilter) { ... }
    
    static HedgingRule of(HedgingRule... hedgingRules) { ... }
    static HedgingRule of(Iterable<? extends HedgingRule> hedgingRules) { ... }
    default HedgingRule orElse(HedgingRule other) { ... }

    CompletionStage<HedgingDecision> shouldContinueHedging(ClientRequestContext ctx, @Nullable Throwable cause);

    default boolean requiresResponseTrailers() { return false; }
}
```
(1): I find it dangerous to define a standard hedging delay (e.g. via `failsafe`) so I would rather force the user to think about a suitable delay for their use case.

## `HedgingRuleBuilder`

```java
public final class HedgingRuleBuilder extends AbstractRuleBuilder<HedgingRuleBuilder> {
    HedgingRuleBuilder(BiPredicate<? super ClientRequestContext, ? super RequestHeaders> requestHeadersFilter) { ... }

    public HedgingRule thenContinueHedging(HedgingDelayProvider hedgingDelayProvider) { ... }

    public HedgingRule thenAbortHedging() { ... }

    static HedgingRule build(BiFunction<? super ClientRequestContext, ? super Throwable, Boolean> ruleFilter,
                           HedgingDecision decision, boolean requiresResponseTrailers) { ... }
}
```

## `HedgingRuleWithContent`
Omitted here but very much like `RetryRuleWithContent` to `RetryRule`.

## `HedgingRuleWithContentBuilder`
Omitted here but very much like `RetryRuleWithContentBuilder` to `RetryRuleWithContent`.

## `HedgingConfig`

```java
public final class HedgingConfig<T extends Response> {

    public static HedgingConfigBuilder<HttpResponse> builder(HedgingRule hedgingRule) { ... }

    public static HedgingConfigBuilder<HttpResponse> builder(
            HedgingRuleWithContent<HttpResponse> hedgingRuleWithContent) { ... }

    public static HedgingConfigBuilder<RpcResponse> builderForRpc(HedgingRule hedgingRule) { ... }

    public static HedgingConfigBuilder<RpcResponse> builderForRpc(
            HedgingRuleWithContent<RpcResponse> hedgingRuleWithContent) { ... }

    public HedgingConfigBuilder<T> toBuilder() { ... }

    public int maxTotalAttempts() { ... }

    public long responseTimeoutMillisForEachAttempt() { ... }

    public HedgingDelayProvider initialHedgingDelayProvider() { ... }

    @Nullable
    public HedgingRule hedgingRule() { ... }

    @Nullable
    public HedgingRuleWithContent<T> hedgingRuleWithContent() { ... }

    public int maxContentLength() { ... }

    public boolean needsContentInRule() { ... }

    public boolean requiresResponseTrailers() { ... }
}
```

## `HedgingConfigBuilder`
Follows naturally from `HedgingConfig`.

## `HedgingClient` or `RetryClient`?

### Option 1: Implement hedging as part of `RetryClient`

#### Required Changes

##### `RetryConfigMapping`

We would need to enable the current `RetryConfigMapping` to map to both, `RetryConfig` and `HedgingConfig`s. For that I would suggest to create a new public, abstract class, `RetryClientConfig<T extends Response>`, from which `RetryConfig` and `HedgingConfig` would inherit. Then I would introduce a new public interface, `RetryClientConfigMapping<T extends Response>` that just manages `RetryClientConfig`s. `RetryClient` would need to differentiate between `RetryConfig` and `HedgingConfig` when it gets the config from the mapping. `RetryConfigMapping` would be deprecated in favor of `RetryClientConfigMapping`.

##### `RetryClient`

`RetryClient` would of course need to respect the possible `HedgingConfig`s that are passed to it via `RetryClientConfigMapping`. Furthermore, it would need to be able to handle the `HedgingDecision` and the `HedgingDelayProvider` in the same way as it currently handles the `RetryDecision` and `Backoff`. I would think of internal separation of the logic to handle the retry policies.

#### Pros
1. No need to create a new client, can easily reuse initialization logic of the client.
2. In gRPC, hedging is presented just as another retry strategy. As such, they respond to the same events as defined [here](https://github.com/grpc/proposal/blob/master/A6-client-retries.md#summary-of-retry-and-hedging-logic). This means that 
    - Users who read the gRPC documentation or spec will find Armeria's implementation more familiar.
3. Most of the time it is a bad idea to retry and hedge a request at the same time as this could dangerously increase the load on the server. By implementing both retry and hedging in the same client, users will provide a single configuration per method and will be less likely to make this mistake (although it is still possible).

#### Cons
- Consolidation bloats the already complex `RetryClient` class. We can expect that we introduce a bunch of `if-else` statements in the `RetryClient` class.
- Users implementing `AbstractRetryingClient` or `RetryConfigMapping` but are only interested in retrying need to deal with hedging too.

### Option 2: Implement as a separate `HedgingClient`

#### Pros
1. Clear separation between hedging and retry logic: allows for further development of hedging without breaking retry users which are probably more numerous than hedging users.
2. Although `RetryClient` can be chained after `HedgingClient` which is questionable, it is possible to chain `HedgingClient` after `RetryClient`. This would allow users to perform retries (perhaps with long backoffs) of hedged requests towards a whole endpoint group. Having a combined client would require the user to integrate two `RetryClient`s in the chain which is ugly for the user as the users now has to think of two `RetryConfig`s which both can either retry or hedge requests.

#### Cons
1. Users that want to retry on one portion of their API and hedge on another portion would have to register two clients and maintain two configurations. To avoid unintentional high load on the server, the would have to pay attention to:
  1. `RetryClient` to be executed before `HedgingClient` in the decorator chain.
  2. The config for `HedgingClient` that it does not overlap with the config of `RetryClient` if they do not want to retry hedged requests. This could get difficult if they are defined in separate places.

From my perspective, issue 1. is less of a problem as most superficial client tests would detect that too many requests are sent. Furthermore the fix for it is easy: just swap the order of the clients in the chain. Issue 2. is more problematic as it is not easy to detect and requires a lot of user discipline to avoid it.




## Benchmarking [WIP]

We should verify the promised latency improvements of hedging via benchmarks. This will also help to find some inefficiencies in the implementation. Details are still to be defined.