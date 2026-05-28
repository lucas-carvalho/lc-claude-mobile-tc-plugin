# Ticket Types — Mobile App QA

The mobile app has four main implementation categories. Each requires a different testing approach.

---

## 1. Functional

**When to use**: The ticket delivers a complete feature already integrated with the real API or with API-mimicking mocks. Focus is on user behavior within the screen.

**Examples**: New screen, onboarding flow, course detail screen, user profile.

**What to test**:

- **Navigation**: How does the user reach this screen? How do they leave? Does the back button work correctly from all entry points?
- **Screen states**: Does the screen have distinct states (e.g. loading, error, empty, populated)? Cover each one.
- **Content variations**: Does the screen change based on data (e.g. course started vs. not started)? Cover each variation as a separate Block.
- **Primary action**: In each state, which element is the main user action? Is it visually highlighted?
- **Secondary actions**: Do disabled buttons appear correctly? Are conditional cards hidden when they should be?
- **Loading and error**: Does the spinner appear while loading? Is an error state shown with a retry option?
- **Mock data**: Are required fields correctly typed (boolean, string, number, array)? Are there mocks for each relevant state?
- **Edge cases**: What happens with missing fields (null/undefined)? Empty arrays? Long strings?

**Block organization (for screens with states)**:
1. Access and Loading (navigation, loading spinner, error state)
2. Structure / Order (visual grouping, sort order)
3. One block per state variation (Not Started, In Progress, Completed, etc.)
4. Card Types / Labels (if applicable)
5. Edge Cases
6. Mock Checklist

---

## 2. Integration

**When to use**: The ticket maps or implements consumption of API endpoints, analytics events, or external SDKs.

**Common technologies**:
- **GraphQL**: Backend queries and mutations
- **Segment**: Analytics events (Click, CTA, Status, Page View)
- **Firebase**: Authentication and analytics
- **RevenueCat**: Subscription management and paywall

**What to test by technology**:

### GraphQL
- Is the correct query/mutation being called (right operation name)?
- Are all required contract fields being sent?
- Are optional variables handled correctly (omitted vs. null)?
- Is the response correctly mapped to UI state?
- Is the loading state shown during the call?
- Are network / API errors handled with an appropriate message?
- Does retry work after an error?

### Segment (Analytics)
- Is the event with the correct name fired at the right moment?
- Are the required event properties present (e.g. `courseId`, `userId`, `status`)?
- Are conditional properties only sent when relevant?
- Is the event **not** fired under incorrect conditions (false positive)?
- Are Page View events fired on screen entry?
- Are CTA events fired on the correct button tap?

### Firebase
- Is the user authenticated before accessing protected content?
- Is the auth token refreshed when expired?
- Does logout / session expiry redirect correctly?
- Are Firebase Analytics events fired correctly (if applicable)?

### RevenueCat
- Is the paywall shown for users without an active subscription?
- Is premium content unlocked after subscription is confirmed?
- Is subscription status (active, expired, trial) correctly reflected in the UI?
- Does "Restore Purchase" work?

**Block organization (integration tickets)**:
1. One block per primary GraphQL endpoint
2. One block for Segment events (if multiple, split by type: Page View, CTA, Status)
3. One block for Firebase (if auth is involved)
4. One block for RevenueCat (if subscriptions are involved)
5. Edge Cases (network failures, expired tokens, missing data)

---

## 3. Component

**When to use**: The ticket delivers a reusable React Native component (button, card, modal, etc.), typically converted to a multi-platform native component.

**What to test**:

- **Visual variants**: Does the component have props that change its appearance (color, size, style)? Cover each relevant variant.
- **States**: Does the component have interactive states (default, pressed, disabled, loading, error)? Test each one.
- **Required vs. optional props**: Does the component work without optional props? Does it show an appropriate fallback?
- **Dynamic content**: Does long text wrap correctly? Do missing images show a placeholder?
- **Accessibility**: Are accessibility labels present? Is contrast adequate? *(Include only if explicitly required by the ticket or client — omit by default.)*
- **Interaction**: Are callbacks (onPress, onChange) fired correctly? Does a disabled component ignore interactions?
- **Multi-platform**: Does the component behave consistently on iOS and Android?

**Block organization**:
1. Visual Variants / Props
2. Interactive States (default, pressed, disabled, loading)
3. Dynamic Content and Edge Cases
4. Accessibility *(only if explicitly in scope for this ticket)*
5. Multi-platform (if the ticket notes platform-specific differences)

---

## 4. Bug

**When to use**: The ticket describes a mapped error, incorrect behavior, or feature blocker.

**What to test**:

- **Reproduction**: Does the original scenario that caused the bug still reproduce before the fix?
- **Fix verification**: After the fix, is the correct behavior shown?
- **Regression**: Does the fix not break adjacent behaviors?
- **Similar cases**: Are there related scenarios that might have the same underlying bug?
- **Fix edge cases**: Does the fix work across all variations of the scenario (different states, different data)?

**Block organization**:
1. Bug Reproduction (original scenario)
2. Fix Verification (expected behavior after the fix)
3. Regression Tests (adjacent features)
4. Variations and Edge Cases (similar situations)

**Important notes for bugs**:
- Always document the reproduction scenario with all required data/state in the Notes column.
- Reference the original bug ticket if one exists.
- If the bug belongs to a category (Functional, Integration, Component), include relevant TCs from that category as well.
