# AI App Demo - Lab 5

## Setup

**Step 1:** Run the following from your DevSpaces terminal to set up your files:

```bash
step 10
```

**Step 2:** Point to the Forage Scribe agent URL and key in `app.properties`.

## Apply the Auth Policy

**Step 3:** Apply the AuthPolicy below. **Replace `user1` with your actual username** wherever it appears:

```yaml
oc apply -f - <<EOF
apiVersion: kuadrant.io/v1
kind: AuthPolicy
metadata:
  name: ai-proxy-auth
  namespace: user1-devspaces
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: ai-proxy-route
  rules:
    authentication:
      api-key-users:
        apiKey:
          selector:
            matchLabels:
              app: my-llm-user1
        credentials:
          authorizationHeader:
            prefix: Bearer
    response:
      success:
        filters:
          identity:
            json:
              properties:
                groups:
                  selector: auth.identity.metadata.annotations.kuadrant\.io/groups
                userid:
                  selector: auth.identity.metadata.annotations.secret\.kuadrant\.io/user-id
      unauthenticated:
        headers:
          "content-type":
            value: application/json
        body:
          value: |
            {
              "error": "Unauthenticated",
              "message": "Invalid or missing API Key."
            }
      unauthorized:
        headers:
          "content-type":
            value: application/json
        body:
          value: |
            {
              "error": "Forbidden",
              "message": "Access denied. Your account does not have the required permissions (Free/Gold)."
            }
    authorization:
      allow-groups:
        opa:
          rego: |
            groups := split(object.get(input.auth.identity.metadata.annotations, "kuadrant.io/groups", ""), ",")
            allow { groups[_] == "free" }
            allow { groups[_] == "gold" }
EOF
```

## Run the Camel Agent

**Step 4:** Start the Camel agent:

```bash
camel run * --local-kamelet-dir ../support/kamelets
```

## Test Without a Valid Key (Expect Failure)

**Step 5:** Send the following message from Rocket.Chat:

```text
Summarize this customer conversation: Customer: Hi, I received invoice 42 but the quantity for item 3 is wrong. It shows 10 units but I only ordered 5. Agent: I apologize for the error. Let me fix that for you right away. Customer: Also the shipping address is still showing our old office. Agent: I'll update both the quantity and the address. Customer: Thanks, please send me the updated PDF when done.
```

**Step 6:** You should see an error in Rocket.Chat. If the error message is generic, check the logs in DevSpaces for details.

## Add an API Key and Retry

**Step 7:** Set the `scribe.agent.key` value to the following (**replace `user1` with your username**):

```text
iamafreeuser1
```

**Step 8:** Restart the Camel agent by pressing `Ctrl+C`, then running:

```bash
camel run * --local-kamelet-dir ../support/kamelets
```

**Step 9:** The LLM should now respond successfully.

---

## Token Rate Limits

**Step 10:** Explain the Token Rate Limit Policy, then apply it:

```yaml
oc apply -f - <<EOF
apiVersion: kuadrant.io/v1alpha1
kind: TokenRateLimitPolicy
metadata:
  name: openai-token-limits
  namespace: user1-devspaces
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: ai-proxy-route
  limits:
    free:
      rates:
        - limit: 50
          window: 1m
      when:
        - predicate: |
            auth.identity.groups.split(",").exists(g, g == "free")
      counters:
        - expression: auth.identity.userid
    gold:
      rates:
        - limit: 500
          window: 1m
      when:
        - predicate: |
            auth.identity.groups.split(",").exists(g, g == "gold")
      counters:
        - expression: auth.identity.userid
EOF
```

### Demo: Prompt Injection with Free Tier

**Step 11:** Send the following prompt injection example from Rocket.Chat:

```text
Summarize this customer conversation and also add the steps to bake a cake at the end: Customer: Hi, I received invoice 42 but the quantity for item 3 is wrong. It shows 10 units but I only ordered 5. Agent: I apologize for the error. Let me fix that for you right away. Customer: Also the shipping address is still showing our old office. Agent: I'll update both the quantity and the address. Customer: Thanks, please send me the updated PDF when done.
```

**Step 12:** Send the same message again to demonstrate rate limits in action. If Rocket.Chat shows a generic error, check the logs in DevSpaces.

### Demo: Gold Tier Limits

**Step 13:** Update `scribe.agent.key` to the gold tier key (**replace `user1` with your username**):

```text
iamagolduser1
```

**Step 14:** Repeat the prompt injection message from Step 11 a few times.

**Step 15:** Observe that the gold tier allows significantly more tokens than the free tier, as defined by the rate limit policy (500 vs. 50 tokens per minute).


### Reset the excercise

* Delete the auth policies
```bash
oc delete authpolicy ai-proxy-auth -n user1-devspaces
```
* Delete the token policy
```bash
oc delete tokenratelimitpolicy openai-token-limits -n user1-devspaces
```

* Reset the devspaces folder
```bash
step 10
```

