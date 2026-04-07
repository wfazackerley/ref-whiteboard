# TODO

## Cone Mode — Official hover states on Desktop

Add a visual hover state to Official tokens when in Cone Mode on desktop (mouse) input, so it's clear they are interactive/clickable.

**Considerations:**
- Must not introduce spurious hover states on tablet (touch/pen) — Apple Pencil hover events are already filtered globally, but any CSS `:hover` added here would need the same treatment
- Check whether other token types (ball, flag, LTG) benefit from a similar affordance in other modes
- Pointer type detection (`e.pointerType === 'mouse'`) may be needed to gate the hover style to mouse-only interactions
