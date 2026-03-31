# Doctor Report: Landmarks App

Real output from running `/swift-assist:doctor` against the [grantiva-examples](https://github.com/grantiva/grantiva-examples) Landmarks app.

```
Accessibility Doctor Report
============================

Scanned: 11 views, 54 interactive elements
Missing identifiers: 54 (0% coverage)

ContentView (5 missing)
  - Tab "Landmarks"           -> content-tab-landmarks (tap by label)
  - Tab "Favorites"           -> content-tab-favorites (tap by label)
  - Tab "Grantiva"            -> content-tab-grantiva (tap by label)
  - Tab "Feedback"            -> content-tab-feedback (tap by label)
  - Tab "Deep Links"          -> content-tab-deeplinks (tap by label)

LandmarkDetailView (3 missing)
  - NavigationLink (category) -> landmark-detail-link-category
  - NavigationLink "Plan Visit" -> landmark-detail-link-plan-visit
  - Button (favorite)         -> landmark-detail-button-favorite

EditLandmarkView (4 missing)
  - TextField "Name"          -> edit-landmark-textfield-name
  - TextField "Location"      -> edit-landmark-textfield-location
  - Button "Cancel"           -> edit-landmark-button-cancel
  - Button "Done"             -> edit-landmark-button-done

FavoritesView (1 missing)
  - Button "Remove" (swipe)   -> favorites-button-remove

VisitConfirmationView (1 missing)
  - Button "Done"             -> visit-confirmation-button-done

ReservationListView (1 missing)
  - Button "Cancel" (swipe)   -> reservation-list-button-cancel

GrantivaView (7 missing)
  - Button "Validate"         -> grantiva-button-validate
  - Button "Refresh"          -> grantiva-button-refresh
  - Button "Clear"            -> grantiva-button-clear
  - Button "Clear Identity"   -> grantiva-button-clear-identity
  - TextField "User ID"       -> grantiva-textfield-user-id
  - Button "Identify"         -> grantiva-button-identify
  - Button "Fetch Flags"      -> grantiva-button-fetch-flags

DeepLinksView (14 missing)
  - NavigationLink "Caching"  -> deeplinks-link-caching-demo
  - Button "Multi Mountains"  -> deeplinks-button-multi-mountains
  - Button "Golden Gate"      -> deeplinks-button-deeplink-golden-gate
  - Button "Lakes"            -> deeplinks-button-deeplink-lakes
  - Button "Show as Sheet"    -> deeplinks-button-show-sheet
  - Button "Show as Full"     -> deeplinks-button-show-fullscreen
  - Button "Edit Landmark"    -> deeplinks-button-edit-landmark
  - Button "Save State"       -> deeplinks-button-save-state
  - Button "Restore State"    -> deeplinks-button-restore-state
  - Button "Clear State"      -> deeplinks-button-clear-state
  - Button "Done" (sheet)     -> deeplinks-button-done-sheet
  - Button "Done" (fullscreen) -> deeplinks-button-done-fullscreen

ServicesDemoView (3 missing)
  - Picker "Cache Policy"     -> services-demo-picker-cache-policy
  - Button "Fetch with Policy" -> services-demo-button-fetch-policy
  - Button "Clear Cache"      -> services-demo-button-clear-cache

Coverage: 0/54 elements have identifiers (0%)
```
