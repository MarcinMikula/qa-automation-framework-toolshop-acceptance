# Iteration 1 — conceptual UI model

This document captures the human-led conceptual model produced during Toolshop reconnaissance before the first test oracle is frozen and before implementation classes are written.

It is a curated acceptance artifact, not raw Playwright Codegen output. Raw Codegen, screenshots, traces, and other discovery material remain outside the repository in the local reconnaissance workspace.

## Context and scope

Target application:

- Practice Software Testing / Toolshop
- `base_url = https://practicesoftwaretesting.com`

Current Iteration 1 slice:

```text
open catalogue
→ enter a search phrase
→ execute search
→ observe results
→ verify results against an evidence-based expected result
```

Iteration 1 deliberately excludes product-details automation, cart, checkout, authentication, API/SOM work, TRE integration, and broader regression coverage.

## Modeling vocabulary used

- **Object** — a representation of a meaningful part of the application used by the automation, for example `Catalog/Search Page`, `ProductCard`, or `ProductPage`.
- **Responsibility** — what that object owns and what should remain inside its boundary.
- **Behavior** — an action or operation the object exposes to its consumer; in implementation these are likely to become methods.
- **State** — information the object currently exposes and that the test can observe or read.

The model is intentionally conceptual. Concrete class names, method names, locator mechanics, return types, and parsing details remain implementation decisions unless explicitly marked otherwise.

## Model relationship

```text
Catalog/Search Page
        │
        │ contains many
        ▼
   ProductCard [*]
        │
        │ can navigate to
        ▼
    ProductPage

ProductPage is recognized by the model but is not implemented in Iteration 1.
```

## Catalog/Search Page

### Responsibilities

- Manage the main catalogue/search view container.
- Expose the interface for entering search criteria and, potentially in later slices, filtering and sorting.
- Aggregate and expose the currently displayed products as a collection of independent `ProductCard` component objects.

### Candidate behaviors

- `search_for_phrase(phrase: str) -> None` — enters the phrase and submits the search by interacting with the search input and submit control.
- `reset_search() -> None` — clears the current search.
- `get_displayed_cards() -> List[ProductCard]` — returns `ProductCard` objects representing the product cards currently visible in the UI.
- `get_results_count_from_caption() -> int` — candidate method for extracting the displayed result count from the search-state caption, for example `Search results for: ... (12)`.

### Candidate observable state

- `is_results_list_empty -> bool` — whether the UI indicates that no results are available.
- `current_search_caption_text -> str` — raw text describing the current search state.

## ProductCard — Component Object candidate

`ProductCard` is a key component candidate for Iteration 1. Reconnaissance showed a repeated, independently meaningful product card with its own root element, product data, navigation target, and additional action.

### Responsibilities

- Represent one product tile in the catalogue by encapsulating a single product-card DOM subtree.
- Expose summary product data directly from the results list without navigating to product details.
- Allow navigation to the selected product's details view.

### Candidate behaviors

- `click_card() -> None` — activates the product-card link and navigates to the corresponding `ProductPage`.
- `add_to_compare() -> None` — triggers the compare action when available on the card.

### Candidate readable data / state

- `id -> str` — product identifier; reconnaissance confirmed that the same identifier is present in both `data-test="product-<id>"` and `href="/product/<id>"`.
- `name -> str` — product name displayed on the card.
- `price` — displayed product price; final representation is not frozen yet.
- `co2_rating -> str` — visible CO2 rating when present.
- product destination / `href` — confirmed as `/product/<id>`.

## ProductPage — Product Details

### Responsibility

- Represent the full product-details view reached after selecting a product from the catalogue.

Reconnaissance confirmed that selecting a card navigates to a distinct route shaped as:

```text
/product/<product-id>
```

and that the details view exposes its own product information and controls.

### Iteration 1 decision

- `Catalog/Search Page`: **IMPLEMENT**
- `ProductCard`: **IMPLEMENT**
- `ProductPage`: **DO NOT IMPLEMENT**

`ProductPage` remains part of the conceptual application model, but the current search-results test slice does not require a product-details class. Discovery does not imply implementation.

## Reconnaissance evidence behind the model

The model was derived from repeated manual exploration using Playwright Codegen plus browser DevTools inspection.

Confirmed observations relevant to this model include:

- catalogue/search and product-details are distinct views;
- selecting a product navigates from the catalogue to `/product/<id>`;
- Codegen recorded the product interaction but did not necessarily emit a separate `goto()` for the application-triggered navigation;
- a complete product card is a semantic anchor element (`<a class="card">`), not only a visual `div`;
- the card root contains `data-test="product-<id>"`;
- the same card contains `href="/product/<id>"`;
- the product card exposes product name, price, CO2 rating, image, and a compare action;
- stable `data-test` attributes are widely available, while some Codegen fallbacks (`nth`, structural selectors, generic text filtering) are weaker locator candidates and should not be copied blindly into the final model;
- related-product links and the main catalogue card may expose different locator shapes, reinforcing the need to model behavior rather than copy Codegen output literally.

## Open implementation decisions

The following points are intentionally not frozen by this conceptual model:

1. **Result-count API** — whether the public contract should remain `get_results_count_from_caption()` or hide the caption parsing detail behind a more semantic `get_results_count()`.
2. **Product-card navigation method name** — whether `click_card()` should remain mechanical or become intention-oriented, for example `open_details()`.
3. **Product ID source** — both `data-test="product-<id>"` and `href="/product/<id>"` contain the identifier; the implementation should choose the cleaner contract rather than assuming one source now.
4. **Price representation** — do not freeze `float` for monetary values. The test may only need visible text; if numeric money operations become necessary, a decimal representation should be considered.
5. **Raw caption exposure** — `current_search_caption_text` may prove useful, but the implementation should expose only the state the test genuinely needs.
6. **Exact first-test oracle** — not frozen yet. The expected result must be established before the implementation is reduced to the minimal set of Page/Component capabilities actually required by the test.

## Acceptance significance

This artifact records the transition from locator discovery to a human-owned automation model:

```text
raw UI / Codegen evidence
→ identify meaningful objects
→ assign responsibilities
→ identify behaviors and observable state
→ decide implementation scope
→ freeze test oracle
→ create minimal implementation design
```

The acceptance question is not whether every discovered UI element can be automated. It is whether the framework and its documentation help a tester turn a real testing need into a small, coherent automation model without leaking application-specific detail into the framework core or implementing unused abstractions prematurely.
