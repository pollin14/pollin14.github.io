## ngIf and ngShow

`ngIf` removes the element of the DOM so if its content has several bindings (watchers) they will not process.
While `ngShow` only hides the element with and `display: none`.

These behavior is similar the the properties `display` and `visibility` with the value `none` and `hidden` respectively.
Both hide the element both `display` hide the element the layout as if it is not in the layout or DOM, while
`visibility` hide the element both the layout is not affect.