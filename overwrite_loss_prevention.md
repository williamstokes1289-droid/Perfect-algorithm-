# overwrite_loss_prevention.py
# OLP — Overwrite-Loss Prevention
#
# Invariant:
#   bool(st.hold_buffer) == st.gap_active
#
# Meaning:
#   - If a SEQ gap is active, we MUST be holding (no mutating transforms).
#   - If we are holding, there MUST be an active gap (hold is not arbitrary).
#
# Guarantee:
#   Prevents geometric decode / mutating paths from overwriting or invalidating
#   incomplete sequence regions. Recovery proceeds via ACCEPT+fallback until the
#   SEQ gap is resolved.

from __future__ import annotations
from dataclasses import dataclass


@dataclass
class StreamState:
    """
    Minimal state model for OLP gating.

    hold_buffer:
        True when we are buffering / deferring mutating transforms to protect
        continuity (write-barrier semantics).
    gap_active:
        True when there is an unresolved SEQ discontinuity.
    require_contiguous:
        If False, SEQ contiguity is not enforced (OLP can be bypassed).
    """
    hold_buffer: bool = False
    gap_active: bool = False
    require_contiguous: bool = True


class OverwriteLossPreventionError(AssertionError):
    """Raised when the OLP invariant is violated."""


def assert_olp_invariant(st: StreamState) -> None:
    """
    Enforce OLP: hold_buffer ⇔ gap_active (when contiguity is required).

    If contiguity isn't required, we do not enforce this invariant.
    """
    if not st.require_contiguous:
        return
    if bool(st.hold_buffer) != bool(st.gap_active):
        raise OverwriteLossPreventionError(
            f"OLP invariant violated: hold_buffer ({st.hold_buffer}) "
            f"must equal gap_active ({st.gap_active})."
        )


def olp_allows_mutation(st: StreamState) -> bool:
    """
    True iff mutating transforms (e.g., geometric decode) are allowed.
    """
    assert_olp_invariant(st)
    return (not st.require_contiguous) or (not st.hold_buffer)


def decode_with_accept_fallback(record: bytes, st: StreamState) -> bytes:
    """
    Example skeleton:
      - If OLP allows mutation, run geometric decode (mutating path).
      - Otherwise ACCEPT+fallback (non-mutating safe path).

    Replace geometric_decode / accept_fallback with your real functions.
    """
    # Enforce the invariant before any decision.
    assert_olp_invariant(st)

    if olp_allows_mutation(st):
        return geometric_decode(record)      # mutating path
    else:
        return accept_fallback(record, st)   # non-mutating safe path


# --- Stubs: replace with your actual implementations --- #

def geometric_decode(record: bytes) -> bytes:
    # Mutating transform example (placeholder)
    return record


def accept_fallback(record: bytes, st: StreamState) -> bytes:
    # Non-mutating safe behavior example (placeholder)
    # e.g., buffer, forward raw, or route to quarantine/repair lane.
    return record


# --- Optional: small self-test --- #

if __name__ == "__main__":
    s = StreamState(hold_buffer=True, gap_active=True, require_contiguous=True)
    assert_olp_invariant(s)  # OK

    s2 = StreamState(hold_buffer=False, gap_active=True, require_contiguous=True)
    try:
        assert_olp_invariant(s2)
    except OverwriteLossPreventionError as e:
        print("Expected violation:", e)
