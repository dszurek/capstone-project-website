# Test Documentation

This document describes the backend test suite located in `server/src/__tests__/`. All tests are written with Jest and Supertest and run against an in-memory MongoDB instance, so no real database or running server is needed.

## Running Tests

All commands should be run from the `server/` directory.

```bash
cd server

# Run the entire test suite
npm test

# Run a single test file
NODE_ENV=test npx jest src/__tests__/<filename>.test.ts

# Run tests in watch mode
npm run test:watch

# Run tests matching a keyword in their describe/it name
NODE_ENV=test npx jest --testNamePattern="Review Creation"
```

## Test Structure

Every integration test file follows the same pattern. A `beforeAll` hook starts an in-memory MongoDB server and seeds the users, profiles, and products the tests need. An `afterEach` hook deletes only the documents that change between tests, so each test starts clean. An `afterAll` hook disconnects from MongoDB and stops the in-memory server.

The shared setup file (`setup.ts`) is used by `authorize.test.ts` only; all other files manage their own lifecycle. JWT tokens are generated directly inside each test file using the same `JWT_SECRET` the app uses (`default_secret` in test mode), so there is no login step required.

Key dependencies used across the suite include `mongodb-memory-server` for the ephemeral in-memory database, `supertest` for issuing HTTP requests to the Express app without a running server, and `ts-jest` for running TypeScript test files directly.

## Test Files

### `authorize.test.ts`

Tests the `authorize(...roles)` middleware that guards protected routes. The test cases cover the middleware calling `next()` when the user's role matches, returning 403 when the role does not match, accepting any one of multiple permitted roles, returning 403 when none of the permitted roles match, and returning 401 with a "log in" message when the auth middleware did not set `req.user`.

### `publicId.test.ts`

Tests the `publicId` utility functions and their integration with the `User` model and the auth API. Public IDs are the anonymous identifiers shown publicly for reviewers, in the format `RXXX-XXXX`.

The unit tests verify that `getPrefixForRole` returns the correct letter for each role, that `generatePublicId` produces IDs matching the expected format, that 50 sequential generations produce no collisions, that the generator retries when an ID is taken and throws after exhausting retries, and that `updatePublicIdPrefix` swaps only the prefix character while preserving the hex portion.

Integration tests on the `User` model confirm that `publicId` is auto-generated on creation without the caller providing it, that new users without a role default to the `R` prefix while users created with `role: 'admin'` receive the `A` prefix, that 20 concurrent user creations produce no collisions, and that updating a field like `firstName` does not overwrite an existing `publicId`.

API tests confirm that `POST /api/auth/register` and `POST /api/auth/login` both return the user's `publicId` in the response. Admin role-change tests on `PUT /api/admin/users/:id/role` confirm that changing a reviewer to an admin updates the prefix from `R` to `A` while keeping the hex portion, that an invalid role such as `superuser` returns 400, and that non-admin callers receive 403. Admin lookup tests on `GET /api/admin/users/by-public-id/:publicId` confirm that a known ID returns the email and `publicId`, and that an unknown ID returns 404.

### `product.test.ts`

Tests the basic product CRUD endpoints. These tests rely on seeded in-memory data and currently use static product IDs (`product-1`), so they serve as smoke tests for the product routes. The cases verify that `GET /api/products` returns a 200 response with a non-empty array, that each product has the expected `name`, `category`, and `price` fields, that `GET /api/products/product-1` returns a single product, that `POST /api/products` creates a product and returns 201 with the submitted name, and that the root route returns the expected API status string.

### `reviewCreation.test.ts`

Tests the `POST /api/reviews` endpoint. The setup creates two reviewers (one approved, one pending) and a product to review. The test cases confirm that an approved reviewer can create a draft (returning 201 with the correct status, reviewer ID, and slug), that a pending reviewer is blocked with a 403 containing the word "approved", that a non-existent product returns 404, that unauthenticated requests return 401, and that duplicate titles receive unique slugs (the second review with the same title gets a `-1` suffix).

### `reviewWorkflow.test.ts`

Tests the full lifecycle of a review from draft through submission, editing, deletion, and public visibility. This is one of the more comprehensive test files.

For the submission workflow at `PATCH /api/reviews/:id/submit`, the tests confirm that a draft transitions to `pending_review` with the `submittedAt` timestamp set, that a review already in `pending_review` cannot be submitted again, that a published review cannot be submitted, and that only the review author can submit (non-owners receive 403).

For resubmission at `PATCH /api/reviews/:id/resubmit`, the tests confirm that a review in `revision_needed` transitions to `pending_review` with the `rejectionReason` cleared, that a draft cannot be resubmitted (it must be submitted first), and that non-owners receive 403.

Status history tracking is verified by confirming that a submission appends one entry to `statusHistory` containing the status, `changedBy`, and `changedAt` fields, that resubmission also appends a history entry, and that a full lifecycle from draft through submit, reject, resubmit, and approve produces four history entries with the correct statuses and notes.

Editing at `PUT /api/reviews/:id` is tested for the cases where the owner can edit a draft, the owner can edit a review in `revision_needed` to address admin feedback, a review in `pending_review` is locked from editing, a published review is also locked, non-owners receive 403, and the `status` field in the request body is ignored so that sending `status: 'published'` does not change the status.

Deletion at `DELETE /api/reviews/:id` is tested for the cases where the owner can delete a draft, the owner can delete a review in `revision_needed`, an admin can delete any draft, published reviews are protected from deletion, reviews in `pending_review` are locked, and non-owner non-admin callers receive 403.

Visibility and retrieval tests confirm that `GET /api/reviews` shows only published reviews to the public, that published reviews are accessible without authentication, that drafts are hidden from unauthenticated users (returning 404), that drafts remain visible to their owner for preview, that drafts are hidden from other reviewers, that admins can see any review regardless of status, and that a non-existent slug returns 404. The `GET /api/reviews/me` endpoint is confirmed to return all statuses for the authenticated reviewer, to exclude other reviewers' content, and to require authentication. The category filter at `GET /api/reviews/category/:slug` is confirmed to return only published reviews, with a non-existent category returning a 200 response with an empty array.

### `reviewQueries.test.ts`

Tests pagination, filtering, sorting, and population on the `GET /api/reviews` listing endpoint and its variants.

Pagination is confirmed for the default page and limit (returning 10 items with correct metadata), a custom `?page=2&limit=2` request (returning the correct slice), and a page beyond the total (returning an empty array rather than an error). The rating filter `?rating=N` returns only reviews with `overall` greater than or equal to N. Sorting by title in descending order places Z before A, and an invalid sort field is ignored without error.

Admin status filtering is verified for `?status=draft` returning only drafts and for the case where an admin without a filter receives all statuses in one response. Population is confirmed to include `name` and `brand` on the product field and `code` and `bio` on the reviewer field.

The `GET /api/reviews/product/:productId` endpoint is tested for returning only published reviews for the given product, rejecting non-ObjectId strings with a 400, returning an empty result for a product with no reviews, and supporting pagination. The `GET /api/reviews/me` endpoint is also tested with pagination and with a status filter that returns only the reviewer's drafts.

### `adminApproval.test.ts`

Tests the admin endpoints for approving and rejecting reviewer profile applications.

For `POST /api/admin/profiles/:id/approve`, the cases verify that an admin can approve a pending profile (status changes to `approved`), that non-pending profiles return 400 with a "cannot approve" message, that a non-existent profile returns 404, that non-admins receive 403, and that unauthenticated requests receive 401.

For `POST /api/admin/profiles/:id/reject`, the cases verify that an admin can reject with a reason (status becomes `rejected` and `statusReason` is stored), that the reason is required (400 when missing from the body), that non-pending profiles cannot be rejected (400 with "cannot reject"), and that non-existent profiles return 404.

### `adminDashboard.test.ts`

Tests the `GET /api/admin/dashboard` endpoint, which returns profile counts and recent activity. The cases confirm that counts per status are returned correctly as `{ pending, approved, rejected, total }`, that all counts are zero when no profiles exist, that recent activity returns profiles updated within the last 30 days, that the activity list is capped at 10 items even when more profiles qualify, that the list is sorted most recent first, and that profiles with `updatedAt` older than 30 days are excluded. Non-admin callers receive 403, and unauthenticated callers receive 401.

### `adminListing.test.ts`

Tests the `GET /api/admin/profiles` endpoint, which lists all reviewer profiles with optional status filtering. The cases confirm that no filter returns all profiles regardless of status, that `?status=pending` and `?status=approved` each return only matching profiles, that an unrecognized status value is ignored and all profiles are returned, that results are sorted newest first, that an empty database returns a 200 response with no results, and that the response includes all demographic fields. Non-admin callers receive 403, and unauthenticated callers receive 401.

### `adminReviewApproval.test.ts`

Tests the admin endpoints for managing review submissions, along with a full end-to-end lifecycle test and dashboard review count statistics.

For `GET /api/admin/reviews`, the cases verify that an admin sees reviews of all statuses by default, that `?status=pending_review` returns only pending reviews, and that non-admin or unauthenticated callers receive 403 or 401 respectively.

For `POST /api/admin/reviews/:id/approve`, the cases verify that a pending review transitions to `published` with `publishDate` set, that attempts to approve a draft or an already-published review return 400, that a non-existent review returns 404, and that non-admins receive 403.

For `POST /api/admin/reviews/:id/reject`, the cases verify that a pending review transitions to `revision_needed` with the `rejectionReason` stored, that the reason is required (400 when missing), that drafts cannot be rejected (400 with "cannot reject"), that non-existent reviews return 404, and that non-admins receive 403.

Status history for admin actions is confirmed by the approve action appending an entry with the `published` status and the reject action appending an entry with the `revision_needed` status and the rejection note.

A single end-to-end test walks through the full lifecycle in sequence: the reviewer creates a draft via `POST /api/reviews`, submits it via `PATCH /api/reviews/:id/submit`, the admin rejects it via `POST /api/admin/reviews/:id/reject`, the reviewer edits the content via `PUT /api/reviews/:id`, resubmits via `PATCH /api/reviews/:id/resubmit`, the admin approves via `POST /api/admin/reviews/:id/approve`, and the review is confirmed to appear on the public `GET /api/reviews` listing.

Dashboard review counts are also verified to include `reviewCounts` with `draft`, `pending_review`, `published`, `revision_needed`, and `total` fields.

### `reviewerProfile.test.ts`

Tests creating, resubmitting, and listing reviewer profiles from the reviewer's perspective.

For `POST /api/reviewers`, the cases confirm that a new profile starts with `pending` status and the correct `userId`, and that a duplicate profile for the same user is blocked with a 400 containing "already exists". Model-level tests confirm that the default status is `pending` without explicit setting, that `statusReason` is stored on the document, and that Mongoose validation rejects non-enum status values.

For `PATCH /api/reviewers/:id/resubmit`, the cases confirm that a rejected profile can resubmit (status returns to `pending` and `statusReason` is cleared), that pending and approved profiles cannot resubmit (400 with "cannot resubmit"), and that non-owners receive 403. The public `GET /api/reviewers` listing is confirmed to show only approved profiles.

### `reviewerProfileMe.test.ts`

Tests the `GET /api/reviewers/me` endpoint, which returns the authenticated reviewer's own profile. The cases confirm that the response includes the populated `firstName` and `lastName` from the User document, that the profile code matches the `REV-XXXXX` format, that a user with no profile receives a 404, and that unauthenticated requests receive 401.

### `reviewerProfileUpdate.test.ts`

Tests the `PUT /api/reviewers/:id` endpoint, where reviewers update their own profile demographics. The cases confirm that allowed fields such as `height`, `bio`, and `age` can be changed, that the `code` field is immutable (sending a new code does not change it while other fields still update), that the `status` field cannot be changed through this endpoint (a request containing `status: 'approved'` leaves the status as `pending`), that the `userId` field is locked, that non-owners receive 403, that unauthenticated requests receive 401, and that a non-existent profile returns 404.

### `reviewerProfileVisibility.test.ts`

Tests the `GET /api/reviewers/:id` visibility rules and the automatic generation of `REV-XXXXX` codes. The cases confirm that the owner sees the full profile with `userId.firstName` and `userId.lastName` populated, that an admin has the same level of access, that other authenticated users see the profile with the `userId` field omitted, and that unauthenticated users also see the anonymous version with the `code` still visible. Sequential code generation is verified by confirming that the first profile receives `REV-00001` and the second receives `REV-00002`.

### `reviewerSearch.test.ts`

Tests the advanced reviewer search endpoint at `GET /api/reviewers/search`. The setup uses 8 seeded profiles with varied demographics to cover the full range of filter combinations.

Default behavior returns only approved profiles (excluding pending and rejected profiles) and includes the pagination metadata fields `page`, `totalPages`, `totalCount`, and `count`.

Free-text search through `?search=` is tested with case-insensitive matches against bio text (the query `"wheelchair"` returns the matching profile), matches against the reviewer code (`"REV-002"` returns the exact profile), and an unknown term returning zero results.

Exact match filters are tested for filtering by `code` (returning a single profile) and by `gender` with case-insensitive matching (returning all three female profiles).

Array membership filters on `vehicleTypes` and `impairmentTypes` are tested for both single-value queries and comma-separated multi-value queries, with the multi-value case matching any of the listed types.

The severity filter at `?impairmentSeverity=severe` returns profiles with at least one severe impairment, and `?impairmentSeverity=mild` returns profiles with at least one mild impairment.

Range filters are tested for age (`?minAge=30&maxAge=50` returns only profiles in that range, and `?minAge=40` alone returns profiles 40 and above), height, weight, and `?minExperience=10` for years of experience.

The status filter returns the single pending profile for `?status=pending` and the single rejected profile for `?status=rejected`. The public visibility filter `?isPublic=true` excludes the one approved non-public profile, while `?isPublic=false` returns only that profile.

Combined filters are tested with gender together with vehicle type and age range (narrowing to a single profile, REV-001), impairment type together with severity (narrowing to REV-003, mobility and severe), and search text together with gender (narrowing to REV-002, where the bio contains "driver" and the gender is female).

Pagination is verified for `?limit=2&page=1` (returning 2 of 6 with `totalPages` of 3), `?limit=2&page=2` (returning the correct second page), and a page beyond the total (returning an empty array rather than an error).

Sorting is verified for `?sortBy=age&sortOrder=asc`, `?sortBy=experienceYears&sortOrder=desc`, and the default sort which falls back to `createdAt` descending.

Edge cases include an empty database returning 200 with `totalCount: 0`, an invalid numeric parameter such as `minAge=invalid` being ignored while all approved profiles are still returned, and regex special characters in the search query being safely escaped without a server error.

### `savedReviews.test.ts`

Tests all four saved-review endpoints for saving, listing, checking, and removing bookmarks.

For `POST /api/users/saved-reviews`, the cases confirm that a published review can be saved (returning 201 with the saved review document), that an optional note is stored and returned, that a duplicate save returns 409, that an unpublished review cannot be saved (404), that a non-existent review returns 404, that unauthenticated requests return 401, and that an invalid `reviewId` returns 400.

For `GET /api/users/saved-reviews`, the cases confirm that the endpoint returns the current user's bookmarks with correct count and pagination metadata, that other users' bookmarks are not returned, that pagination parameters work, and that archived reviews are filtered out of results while the bookmark record itself is preserved.

For `GET /api/users/saved-reviews/check/:reviewId`, the cases confirm that a bookmarked review returns `saved: true` along with the `savedReviewId`, that an unbookmarked review returns `saved: false` without error, and that an invalid `reviewId` returns 400.

For `DELETE /api/users/saved-reviews/:id`, the cases confirm that the owner can remove their bookmark and the document is deleted from the database, that a non-owner receives 404 rather than any indication the bookmark exists, that a non-existent bookmark returns 404, and that unauthenticated requests return 401.

## Summary

| File | Type | Coverage |
|------|------|----------|
| `authorize.test.ts` | Unit | Role-based access middleware |
| `publicId.test.ts` | Unit and Integration | Anonymous ID generation and user model |
| `product.test.ts` | Integration | Product CRUD routes |
| `reviewCreation.test.ts` | Integration | Creating new reviews |
| `reviewWorkflow.test.ts` | Integration | Review status lifecycle, editing, deletion, visibility |
| `reviewQueries.test.ts` | Integration | Pagination, filtering, sorting on review listings |
| `adminApproval.test.ts` | Integration | Admin approving and rejecting reviewer profiles |
| `adminDashboard.test.ts` | Integration | Admin dashboard stats and recent activity |
| `adminListing.test.ts` | Integration | Admin profile listing with filters |
| `adminReviewApproval.test.ts` | Integration | Admin review management and end-to-end lifecycle |
| `reviewerProfile.test.ts` | Integration | Profile creation, resubmission, status model |
| `reviewerProfileMe.test.ts` | Integration | Authenticated user's own profile endpoint |
| `reviewerProfileUpdate.test.ts` | Integration | Profile editing with field allowlist |
| `reviewerProfileVisibility.test.ts` | Integration | Profile privacy rules and code auto-generation |
| `reviewerSearch.test.ts` | Integration | Advanced reviewer search with all filters |
| `savedReviews.test.ts` | Integration | Bookmark save, list, check, and delete |
