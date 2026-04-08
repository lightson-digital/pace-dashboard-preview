# Pace Dashboard v5 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an automated pace dashboard (v5) that generates a single HTML file per hotel from Supabase snapshot data, replacing manual Excel pace reports.

**Architecture:** Python queries Supabase → builds JSON payload → renders Jinja2 template → outputs a self-contained HTML file with embedded data and vanilla JS interactivity. All comparison modes and OOO variants are pre-computed server-side.

**Tech Stack:** Python 3.9.6, Supabase (PostgreSQL), Jinja2, vanilla JS, inline SVG

**Working directories:**
- Pipeline: `lights-on/revenue-reporting-automation/pace-pipeline/` (referred to as `pace-pipeline/` below)
- Dashboard preview: `lights-on/pace-dashboard-preview/`

**Design spec:** `lights-on/pace-dashboard-preview/docs/specs/DESIGN.md`

---

## File Map

### New files

| File | Responsibility |
|------|---------------|
| `pace-pipeline/src/dashboard/__init__.py` | Package marker |
| `pace-pipeline/src/dashboard/snapshot_finder.py` | Find comparison snapshot dates with WoW/STLY fallback logic |
| `pace-pipeline/src/dashboard/payload.py` | Build the complete DATA JSON structure from query results |
| `pace-pipeline/src/dashboard/charts.py` | Generate hero pacing chart as inline SVG string |
| `pace-pipeline/src/dashboard/template.html` | Jinja2 template — full HTML/CSS/JS for the dashboard |
| `pace-pipeline/src/dashboard/generate.py` | CLI entry point: query → payload → render → write HTML |
| `pace-pipeline/tests/test_snapshot_finder.py` | Unit tests for snapshot date lookup + fallback |
| `pace-pipeline/tests/test_payload.py` | Unit tests for payload construction + null handling |
| `pace-pipeline/tests/test_charts.py` | Unit tests for SVG chart generation |
| `pace-pipeline/tests/test_dashboard_queries.py` | Unit tests for new query functions |

### Modified files

| File | Changes |
|------|---------|
| `pace-pipeline/src/queries.py` | Add `rolling_12_month_window()`, extend `monthly_aggregate_v2()` with per-day OOO occupancy + null handling |

---

## Task 1: Extend Query Layer — Rolling 12-Month with OOO

Add a new `monthly_aggregate_v2()` function that computes per-day OOO-aware occupancy and returns both OOO variants. Keep the original `monthly_aggregate()` untouched for backward compatibility.

**Files:**
- Modify: `pace-pipeline/src/queries.py`
- Create: `pace-pipeline/tests/test_dashboard_queries.py`

- [ ] **Step 1: Write failing tests for rolling_12_month_window**

```python
# tests/test_dashboard_queries.py
from __future__ import annotations

from datetime import date

import pytest


def test_rolling_12_month_window_april():
    from src.queries import rolling_12_month_window
    start, end = rolling_12_month_window(date(2026, 4, 5))
    assert start == date(2026, 4, 1)
    assert end == date(2027, 3, 31)


def test_rolling_12_month_window_january():
    from src.queries import rolling_12_month_window
    start, end = rolling_12_month_window(date(2026, 1, 15))
    assert start == date(2026, 1, 1)
    assert end == date(2026, 12, 31)


def test_rolling_12_month_window_december():
    from src.queries import rolling_12_month_window
    start, end = rolling_12_month_window(date(2026, 12, 20))
    assert start == date(2026, 12, 1)
    assert end == date(2027, 11, 30)
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline && python -m pytest tests/test_dashboard_queries.py -v -m "not integration"`
Expected: FAIL — `ImportError: cannot import name 'rolling_12_month_window'`

- [ ] **Step 3: Implement rolling_12_month_window**

Add to `pace-pipeline/src/queries.py`:

```python
def rolling_12_month_window(snapshot_date: date) -> Tuple[date, date]:
    """Compute rolling 12-month window: first day of snapshot's month + 11 months.

    Returns (start, end) where:
        start = first day of snapshot_date's month
        end = last day of month + 11
    """
    start = snapshot_date.replace(day=1)
    # End month = start month + 11
    end_month = start.month + 11
    end_year = start.year
    while end_month > 12:
        end_month -= 12
        end_year += 1
    # Last day of end_month
    if end_month == 12:
        end = date(end_year + 1, 1, 1)
    else:
        end = date(end_year, end_month + 1, 1)
    from datetime import timedelta
    end = end - timedelta(days=1)
    return start, end
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline && python -m pytest tests/test_dashboard_queries.py -v -m "not integration"`
Expected: 3 PASSED

- [ ] **Step 5: Write failing tests for monthly_aggregate_v2**

Add to `tests/test_dashboard_queries.py`:

```python
from decimal import Decimal
from unittest.mock import patch, MagicMock
from src.models import HotelConfig


def _make_hotel(total_rooms: int = 100, occ_excludes_ooo: bool = False) -> HotelConfig:
    return HotelConfig(
        id='TEST',
        name='Test Hotel',
        total_rooms=total_rooms,
        pms_type='stayntouch',
        timezone='Pacific/Honolulu',
        email_alias='test@test.com',
        gdrive_otb_folder_id=None,
        gdrive_forecast_folder_id=None,
        active=True,
    )


def _make_row(stay_date: str, rooms_sold: int, total_revenue: str,
              room_revenue: str = None, available_rooms: int = 80,
              ooo_rooms: int = 0) -> dict:
    row = {
        'stay_date': stay_date,
        'rooms_sold': rooms_sold,
        'total_revenue': total_revenue,
        'room_revenue': room_revenue,
        'available_rooms': available_rooms,
        'ooo_rooms': ooo_rooms,
    }
    return row


@patch('src.queries._fetch_snapshot_rows')
@patch('src.queries.get_hotel')
@patch('src.queries.get_client')
def test_monthly_aggregate_v2_basic(mock_client, mock_get_hotel, mock_fetch):
    from src.queries import monthly_aggregate_v2
    mock_get_hotel.return_value = _make_hotel(total_rooms=100)
    mock_fetch.return_value = [
        _make_row('2026-04-01', 80, '44000.00', '44000.00', 10, 10),
        _make_row('2026-04-02', 90, '49500.00', '49500.00', 5, 5),
    ]
    result = monthly_aggregate_v2('TEST', date(2026, 4, 5))
    assert len(result) == 1
    month = result[0]
    assert month['month'] == '2026-04'
    assert month['rooms'] == 170
    assert month['revenue'] == Decimal('93500.00')
    # ADR = 93500 / 170
    assert month['adr'] == Decimal('93500.00') / 170
    # Excl OOO: denom = (80+10) + (90+5) = 185 (rooms_sold + available for each day)
    assert month['occ_excl_ooo'] is not None
    # Incl OOO: denom = (80+10+10) + (90+5+5) = 200
    assert month['occ_incl_ooo'] is not None


@patch('src.queries._fetch_snapshot_rows')
@patch('src.queries.get_hotel')
@patch('src.queries.get_client')
def test_monthly_aggregate_v2_zero_denominator(mock_client, mock_get_hotel, mock_fetch):
    from src.queries import monthly_aggregate_v2
    mock_get_hotel.return_value = _make_hotel(total_rooms=100, occ_excludes_ooo=True)
    # All rooms OOO — excl_ooo denominator is 0
    mock_fetch.return_value = [
        _make_row('2026-04-01', 0, '0', '0', 0, 100),
    ]
    result = monthly_aggregate_v2('TEST', date(2026, 4, 5))
    month = result[0]
    assert month['adr'] is None  # rooms_sold = 0
    assert month['occ_excl_ooo'] is None  # denominator = 0
    assert month['occ_incl_ooo'] is not None  # denominator = 100 (total)
    assert month['revpar_excl_ooo'] is None
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline && python -m pytest tests/test_dashboard_queries.py::test_monthly_aggregate_v2_basic tests/test_dashboard_queries.py::test_monthly_aggregate_v2_zero_denominator -v -m "not integration"`
Expected: FAIL — `ImportError: cannot import name 'monthly_aggregate_v2'`

- [ ] **Step 7: Implement monthly_aggregate_v2**

Add to `pace-pipeline/src/queries.py`:

```python
def monthly_aggregate_v2(
    hotel_id: str,
    snapshot_date: date,
    snapshot_type: str = 'otb',
) -> List[Dict]:
    """Compute monthly aggregation with per-day OOO-aware occupancy.

    Returns list of dicts with keys:
        month, label, rooms, revenue, adr,
        occ_excl_ooo, occ_incl_ooo,
        revpar_excl_ooo, revpar_incl_ooo,
        available_room_nights, ooo_room_nights

    Null contract:
        - adr is None when rooms_sold = 0
        - occ_excl_ooo / revpar_excl_ooo is None when excl_ooo denominator = 0
        - occ_incl_ooo / revpar_incl_ooo is None when incl_ooo denominator = 0
    """
    client = get_client()
    hotel = get_hotel(client, hotel_id)

    rows = _fetch_snapshot_rows(hotel_id, snapshot_date, snapshot_type)

    monthly = defaultdict(lambda: {
        'rooms_sold': 0,
        'revenue': Decimal('0'),
        'room_revenue': Decimal('0'),
        'denom_excl_ooo': 0,  # sum of (rooms_sold + available_rooms) per day
        'denom_incl_ooo': 0,  # sum of (rooms_sold + available_rooms + ooo_rooms) per day
        'ooo_room_nights': 0,
        'days': 0,
    })

    for row in rows:
        stay_date = date.fromisoformat(row['stay_date'])
        stay_month = stay_date.replace(day=1)
        key = stay_month

        rooms_sold = int(row['rooms_sold'] or 0)
        revenue = Decimal(str(row['total_revenue'] or 0))
        room_rev = row.get('room_revenue')
        room_revenue = Decimal(str(room_rev)) if room_rev is not None else revenue

        available = int(row.get('available_rooms') or 0)
        ooo = int(row.get('ooo_rooms') or 0)

        # Per-day denominators from actual snapshot data
        day_denom_excl = rooms_sold + available  # total - ooo
        day_denom_incl = rooms_sold + available + ooo  # total inventory

        monthly[key]['rooms_sold'] += rooms_sold
        monthly[key]['revenue'] += revenue
        monthly[key]['room_revenue'] += room_revenue
        monthly[key]['denom_excl_ooo'] += day_denom_excl
        monthly[key]['denom_incl_ooo'] += day_denom_incl
        monthly[key]['ooo_room_nights'] += ooo
        monthly[key]['days'] += 1

    results = []
    for stay_month in sorted(monthly.keys()):
        agg = monthly[stay_month]
        rooms_sold = agg['rooms_sold']
        room_revenue = agg['room_revenue']
        revenue = agg['revenue']
        denom_excl = agg['denom_excl_ooo']
        denom_incl = agg['denom_incl_ooo']

        # ADR: null when no rooms sold
        adr = room_revenue / rooms_sold if rooms_sold > 0 else None

        # Occupancy: null when denominator is zero
        if denom_excl > 0:
            occ_excl = float(Decimal(str(rooms_sold)) / Decimal(str(denom_excl)) * 100)
            revpar_excl = float(revenue / Decimal(str(denom_excl)))
        else:
            occ_excl = None
            revpar_excl = None

        if denom_incl > 0:
            occ_incl = float(Decimal(str(rooms_sold)) / Decimal(str(denom_incl)) * 100)
            revpar_incl = float(revenue / Decimal(str(denom_incl)))
        else:
            occ_incl = None
            revpar_incl = None

        month_str = stay_month.strftime('%Y-%m')
        label = stay_month.strftime('%b %Y')

        results.append({
            'month': month_str,
            'label': label,
            'rooms': rooms_sold,
            'revenue': revenue,
            'adr': float(adr) if adr is not None else None,
            'occ_excl_ooo': round(occ_excl, 2) if occ_excl is not None else None,
            'occ_incl_ooo': round(occ_incl, 2) if occ_incl is not None else None,
            'revpar_excl_ooo': round(revpar_excl, 2) if revpar_excl is not None else None,
            'revpar_incl_ooo': round(revpar_incl, 2) if revpar_incl is not None else None,
            'available_room_nights': denom_incl,
            'ooo_room_nights': agg['ooo_room_nights'],
        })

    return results
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline && python -m pytest tests/test_dashboard_queries.py -v -m "not integration"`
Expected: 5 PASSED

- [ ] **Step 9: Commit**

```bash
cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline
git add src/queries.py tests/test_dashboard_queries.py
git commit -m "feat(queries): add rolling_12_month_window and monthly_aggregate_v2 with OOO handling"
```

---

## Task 2: Snapshot Finder — Comparison Date Lookup with Fallback

Find the comparison snapshot date for WoW and STLY modes, with the fallback logic specified in the design (WoW: ±1 day, STLY: ±3 days).

**Files:**
- Create: `pace-pipeline/src/dashboard/__init__.py`
- Create: `pace-pipeline/src/dashboard/snapshot_finder.py`
- Create: `pace-pipeline/tests/test_snapshot_finder.py`

- [ ] **Step 1: Create package marker**

```bash
mkdir -p /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline/src/dashboard
touch /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline/src/dashboard/__init__.py
```

- [ ] **Step 2: Write failing tests**

```python
# tests/test_snapshot_finder.py
from __future__ import annotations

from datetime import date
from unittest.mock import patch

import pytest


def _mock_snapshot_dates(*available_dates):
    """Return a mock that says these snapshot_dates exist."""
    available = set(available_dates)

    def mock_has_snapshot(hotel_id: str, snapshot_date: date, snapshot_type: str) -> bool:
        return snapshot_date in available

    return mock_has_snapshot


class TestFindWoWSnapshot:
    @patch('src.dashboard.snapshot_finder._has_snapshot')
    def test_exact_match(self, mock_has):
        from src.dashboard.snapshot_finder import find_comparison_snapshot
        mock_has.side_effect = _mock_snapshot_dates(date(2026, 3, 29))
        result = find_comparison_snapshot('KAH', date(2026, 4, 5), 'wow')
        assert result == date(2026, 3, 29)

    @patch('src.dashboard.snapshot_finder._has_snapshot')
    def test_fallback_minus_6(self, mock_has):
        from src.dashboard.snapshot_finder import find_comparison_snapshot
        # Exact -7 missing, try -6
        mock_has.side_effect = _mock_snapshot_dates(date(2026, 3, 30))
        result = find_comparison_snapshot('KAH', date(2026, 4, 5), 'wow')
        assert result == date(2026, 3, 30)

    @patch('src.dashboard.snapshot_finder._has_snapshot')
    def test_fallback_minus_8(self, mock_has):
        from src.dashboard.snapshot_finder import find_comparison_snapshot
        # Exact -7 and -6 missing, try -8
        mock_has.side_effect = _mock_snapshot_dates(date(2026, 3, 28))
        result = find_comparison_snapshot('KAH', date(2026, 4, 5), 'wow')
        assert result == date(2026, 3, 28)

    @patch('src.dashboard.snapshot_finder._has_snapshot')
    def test_no_fallback_returns_none(self, mock_has):
        from src.dashboard.snapshot_finder import find_comparison_snapshot
        mock_has.side_effect = _mock_snapshot_dates()  # nothing available
        result = find_comparison_snapshot('KAH', date(2026, 4, 5), 'wow')
        assert result is None


class TestFindSTLYSnapshot:
    @patch('src.dashboard.snapshot_finder._has_snapshot')
    def test_exact_match(self, mock_has):
        from src.dashboard.snapshot_finder import find_comparison_snapshot
        mock_has.side_effect = _mock_snapshot_dates(date(2025, 4, 7))
        result = find_comparison_snapshot('KAH', date(2026, 4, 5), 'stly')
        # 364 days prior = 2025-04-07
        assert result == date(2025, 4, 7)

    @patch('src.dashboard.snapshot_finder._has_snapshot')
    def test_fallback_within_3_days(self, mock_has):
        from src.dashboard.snapshot_finder import find_comparison_snapshot
        # Only 361 days prior available
        mock_has.side_effect = _mock_snapshot_dates(date(2025, 4, 10))
        result = find_comparison_snapshot('KAH', date(2026, 4, 5), 'stly')
        assert result == date(2025, 4, 10)

    @patch('src.dashboard.snapshot_finder._has_snapshot')
    def test_no_fallback_returns_none(self, mock_has):
        from src.dashboard.snapshot_finder import find_comparison_snapshot
        mock_has.side_effect = _mock_snapshot_dates()
        result = find_comparison_snapshot('KAH', date(2026, 4, 5), 'stly')
        assert result is None


class TestFindBudgetSnapshot:
    @patch('src.dashboard.snapshot_finder._has_snapshot')
    def test_always_returns_none(self, mock_has):
        from src.dashboard.snapshot_finder import find_comparison_snapshot
        result = find_comparison_snapshot('KAH', date(2026, 4, 5), 'budget')
        assert result is None
        mock_has.assert_not_called()
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline && python -m pytest tests/test_snapshot_finder.py -v -m "not integration"`
Expected: FAIL — `ModuleNotFoundError`

- [ ] **Step 4: Implement snapshot_finder.py**

```python
# src/dashboard/snapshot_finder.py
from __future__ import annotations

from datetime import date, timedelta
from typing import Optional

from src.db import get_client


def _has_snapshot(hotel_id: str, snapshot_date: date, snapshot_type: str) -> bool:
    """Check if a snapshot exists for the given hotel/date/type."""
    client = get_client()
    result = (
        client.table('snapshots')
        .select('id', count='exact')
        .eq('hotel_id', hotel_id)
        .eq('snapshot_date', snapshot_date.isoformat())
        .eq('snapshot_type', snapshot_type)
        .limit(1)
        .execute()
    )
    return result.count > 0


def find_comparison_snapshot(
    hotel_id: str,
    snapshot_date: date,
    mode: str,
    snapshot_type: str = 'otb',
) -> Optional[date]:
    """Find comparison snapshot date with fallback logic.

    Args:
        hotel_id: Hotel code (e.g., 'KAH')
        snapshot_date: The present snapshot date
        mode: 'wow' (7 days, ±1 fallback), 'stly' (364 days, ±3 fallback), or 'budget'
        snapshot_type: Snapshot type (default 'otb')

    Returns:
        The comparison snapshot date, or None if not found.
    """
    if mode == 'budget':
        # Budget comparison not yet supported
        return None

    if mode == 'wow':
        base_offset = 7
        max_fallback = 1  # try -7, -6, -8
    elif mode == 'stly':
        base_offset = 364  # 52 weeks
        max_fallback = 3  # try -364, -363, -365, -362, -366, -361, -367
    else:
        return None

    target = snapshot_date - timedelta(days=base_offset)

    # Check exact match first
    if _has_snapshot(hotel_id, target, snapshot_type):
        return target

    # Try fallback: alternate +1, -1, +2, -2, etc.
    for offset in range(1, max_fallback + 1):
        candidate_plus = target + timedelta(days=offset)
        if _has_snapshot(hotel_id, candidate_plus, snapshot_type):
            return candidate_plus
        candidate_minus = target - timedelta(days=offset)
        if _has_snapshot(hotel_id, candidate_minus, snapshot_type):
            return candidate_minus

    return None
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline && python -m pytest tests/test_snapshot_finder.py -v -m "not integration"`
Expected: 7 PASSED

- [ ] **Step 6: Commit**

```bash
cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline
git add src/dashboard/__init__.py src/dashboard/snapshot_finder.py tests/test_snapshot_finder.py
git commit -m "feat(dashboard): add snapshot finder with WoW/STLY fallback logic"
```

---

## Task 3: Payload Builder — Assemble the DATA JSON

Build the complete JSON payload structure that gets embedded in the HTML. This is the bridge between queries and the template.

**Files:**
- Create: `pace-pipeline/src/dashboard/payload.py`
- Create: `pace-pipeline/tests/test_payload.py`

- [ ] **Step 1: Write failing tests for _compute_pickup**

```python
# tests/test_payload.py
from __future__ import annotations

from datetime import date
from decimal import Decimal

import pytest


class TestAlignStlyDate:
    def test_normal_date(self):
        from src.dashboard.payload import _align_stly_date
        result = _align_stly_date(date(2025, 4, 7), 1)
        assert result == date(2026, 4, 7)

    def test_leap_day_to_non_leap_year(self):
        from src.dashboard.payload import _align_stly_date
        result = _align_stly_date(date(2024, 2, 29), 1)
        assert result == date(2025, 2, 28)

    def test_leap_day_to_leap_year(self):
        from src.dashboard.payload import _align_stly_date
        result = _align_stly_date(date(2024, 2, 29), 4)
        assert result == date(2028, 2, 29)

    def test_negative_offset(self):
        from src.dashboard.payload import _align_stly_date
        result = _align_stly_date(date(2026, 3, 15), -1)
        assert result == date(2025, 3, 15)


class TestComputePickup:
    def test_basic_pickup(self):
        from src.dashboard.payload import _compute_pickup
        current = {'rooms': 100, 'revenue': Decimal('55000')}
        comparison = {'rooms': 80, 'revenue': Decimal('40000')}
        result = _compute_pickup(current, comparison)
        assert result['rooms'] == 20
        assert result['revenue'] == Decimal('15000')
        assert result['adr'] == 750.0  # 15000 / 20

    def test_zero_pickup_rooms(self):
        from src.dashboard.payload import _compute_pickup
        current = {'rooms': 80, 'revenue': Decimal('40000')}
        comparison = {'rooms': 80, 'revenue': Decimal('40000')}
        result = _compute_pickup(current, comparison)
        assert result['rooms'] == 0
        assert result['adr'] is None

    def test_negative_pickup(self):
        from src.dashboard.payload import _compute_pickup
        current = {'rooms': 70, 'revenue': Decimal('35000')}
        comparison = {'rooms': 80, 'revenue': Decimal('40000')}
        result = _compute_pickup(current, comparison)
        assert result['rooms'] == -10
        assert result['revenue'] == Decimal('-5000')
        assert result['adr'] == 500.0  # -5000 / -10


class TestComputeDelta:
    def test_basic_delta(self):
        from src.dashboard.payload import _compute_delta
        result = _compute_delta(100.0, 80.0)
        assert result['delta'] == 20.0
        assert result['delta_pct'] == 25.0

    def test_null_current(self):
        from src.dashboard.payload import _compute_delta
        result = _compute_delta(None, 80.0)
        assert result['delta'] is None
        assert result['delta_pct'] is None

    def test_null_comparison(self):
        from src.dashboard.payload import _compute_delta
        result = _compute_delta(100.0, None)
        assert result['delta'] is None
        assert result['delta_pct'] is None

    def test_zero_comparison(self):
        from src.dashboard.payload import _compute_delta
        result = _compute_delta(100.0, 0.0)
        assert result['delta'] == 100.0
        assert result['delta_pct'] is None  # can't divide by zero


class TestBuildKpiBanner:
    def test_aggregates_12_months(self):
        from src.dashboard.payload import _build_kpi_banner
        current_months = [
            {'rooms': 100, 'revenue': Decimal('50000'), 'adr': 500.0,
             'occ_excl_ooo': 45.0, 'occ_incl_ooo': 40.0,
             'revpar_excl_ooo': 225.0, 'revpar_incl_ooo': 200.0,
             'available_room_nights': 3000, 'ooo_room_nights': 300},
            {'rooms': 120, 'revenue': Decimal('66000'), 'adr': 550.0,
             'occ_excl_ooo': 50.0, 'occ_incl_ooo': 44.0,
             'revpar_excl_ooo': 275.0, 'revpar_incl_ooo': 242.0,
             'available_room_nights': 3000, 'ooo_room_nights': 300},
        ]
        comparison_months = [
            {'rooms': 90, 'revenue': Decimal('45000'), 'adr': 500.0,
             'occ_excl_ooo': 40.0, 'occ_incl_ooo': 36.0,
             'revpar_excl_ooo': 200.0, 'revpar_incl_ooo': 180.0,
             'available_room_nights': 3000, 'ooo_room_nights': 300},
            {'rooms': 110, 'revenue': Decimal('60500'), 'adr': 550.0,
             'occ_excl_ooo': 48.0, 'occ_incl_ooo': 42.0,
             'revpar_excl_ooo': 264.0, 'revpar_incl_ooo': 231.0,
             'available_room_nights': 3000, 'ooo_room_nights': 300},
        ]
        result = _build_kpi_banner(current_months, comparison_months)
        # Total rooms: 220 current, 200 comparison
        assert result['revenue']['current'] == 116000.0
        assert result['revenue']['comparison'] == 105500.0
        # ADR = total_revenue / total_rooms
        assert round(result['adr']['current'], 2) == round(116000 / 220, 2)

    def test_null_month_excluded_from_denominator(self):
        from src.dashboard.payload import _build_kpi_banner
        current_months = [
            {'rooms': 0, 'revenue': Decimal('0'), 'adr': None,
             'occ_excl_ooo': None, 'occ_incl_ooo': None,
             'revpar_excl_ooo': None, 'revpar_incl_ooo': None,
             'available_room_nights': 0, 'ooo_room_nights': 0},
        ]
        result = _build_kpi_banner(current_months, None)
        assert result['adr']['current'] is None
        assert result['occupancy_excl_ooo']['current'] is None
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline && python -m pytest tests/test_payload.py -v -m "not integration"`
Expected: FAIL — `ModuleNotFoundError`

- [ ] **Step 3: Implement payload.py**

```python
# src/dashboard/payload.py
from __future__ import annotations

from datetime import date
from decimal import Decimal
from typing import Dict, List, Optional, Tuple


def _align_stly_date(d: date, year_offset: int) -> Optional[date]:
    """Shift a date by year_offset years, handling leap-day safely.

    Feb 29 in a leap year maps to Feb 28 in a non-leap year.
    Returns None only if the date is fundamentally unmappable (shouldn't happen
    with the Feb 28 fallback, but defensive).
    """
    target_year = d.year + year_offset
    try:
        return d.replace(year=target_year)
    except ValueError:
        # Feb 29 -> non-leap year: fall back to Feb 28
        if d.month == 2 and d.day == 29:
            return date(target_year, 2, 28)
        return None


def _compute_pickup(current: Dict, comparison: Dict) -> Dict:
    """Compute pickup between current and comparison month/day data."""
    pickup_rooms = current['rooms'] - comparison['rooms']
    pickup_revenue = current['revenue'] - comparison['revenue']
    if pickup_rooms != 0:
        pickup_adr = round(float(pickup_revenue / pickup_rooms), 2)
    else:
        pickup_adr = None
    return {
        'rooms': pickup_rooms,
        'revenue': pickup_revenue,
        'adr': pickup_adr,
    }


def _compute_delta(current, comparison):
    # type: (Optional[float], Optional[float]) -> Dict
    """Compute delta and delta_pct between two values. Null-safe."""
    if current is None or comparison is None:
        return {'delta': None, 'delta_pct': None}
    delta = current - comparison
    if comparison != 0:
        delta_pct = round(delta / comparison * 100, 2)
    else:
        delta_pct = None
    return {'delta': round(delta, 2), 'delta_pct': delta_pct}


def _safe_float(val):
    # type: (Optional[Decimal]) -> Optional[float]
    """Convert Decimal to float, preserving None."""
    if val is None:
        return None
    return float(val)


def _build_kpi_banner(
    current_months: List[Dict],
    comparison_months: Optional[List[Dict]],
) -> Dict:
    """Aggregate rolling 12-month KPIs from monthly data.

    Excludes null months from denominator calculations.
    """
    # Aggregate current
    total_rooms = sum(m['rooms'] for m in current_months)
    total_revenue = sum(m['revenue'] for m in current_months)
    # Denominators: sum only from months with non-null occupancy
    denom_excl = sum(
        m['available_room_nights'] - m['ooo_room_nights']
        for m in current_months
        if m['occ_excl_ooo'] is not None
    )
    denom_incl = sum(
        m['available_room_nights']
        for m in current_months
        if m['occ_incl_ooo'] is not None
    )

    current_revenue = _safe_float(total_revenue)
    current_adr = round(float(total_revenue / total_rooms), 2) if total_rooms > 0 else None
    current_occ_excl = round(total_rooms / denom_excl * 100, 2) if denom_excl > 0 else None
    current_occ_incl = round(total_rooms / denom_incl * 100, 2) if denom_incl > 0 else None
    current_revpar_excl = round(float(total_revenue) / denom_excl, 2) if denom_excl > 0 else None
    current_revpar_incl = round(float(total_revenue) / denom_incl, 2) if denom_incl > 0 else None

    # Aggregate comparison (if available)
    if comparison_months:
        comp_rooms = sum(m['rooms'] for m in comparison_months)
        comp_revenue = sum(m['revenue'] for m in comparison_months)
        comp_denom_excl = sum(
            m['available_room_nights'] - m['ooo_room_nights']
            for m in comparison_months
            if m['occ_excl_ooo'] is not None
        )
        comp_denom_incl = sum(
            m['available_room_nights']
            for m in comparison_months
            if m['occ_incl_ooo'] is not None
        )
        comp_revenue_f = _safe_float(comp_revenue)
        comp_adr = round(float(comp_revenue / comp_rooms), 2) if comp_rooms > 0 else None
        comp_occ_excl = round(comp_rooms / comp_denom_excl * 100, 2) if comp_denom_excl > 0 else None
        comp_occ_incl = round(comp_rooms / comp_denom_incl * 100, 2) if comp_denom_incl > 0 else None
        comp_revpar_excl = round(float(comp_revenue) / comp_denom_excl, 2) if comp_denom_excl > 0 else None
        comp_revpar_incl = round(float(comp_revenue) / comp_denom_incl, 2) if comp_denom_incl > 0 else None
    else:
        comp_revenue_f = None
        comp_adr = None
        comp_occ_excl = None
        comp_occ_incl = None
        comp_revpar_excl = None
        comp_revpar_incl = None

    return {
        'revenue': {
            'current': current_revenue,
            'comparison': comp_revenue_f,
            **_compute_delta(current_revenue, comp_revenue_f),
        },
        'adr': {
            'current': current_adr,
            'comparison': comp_adr,
            **_compute_delta(current_adr, comp_adr),
        },
        'occupancy_excl_ooo': {
            'current': current_occ_excl,
            'comparison': comp_occ_excl,
            **_compute_delta(current_occ_excl, comp_occ_excl),
        },
        'occupancy_incl_ooo': {
            'current': current_occ_incl,
            'comparison': comp_occ_incl,
            **_compute_delta(current_occ_incl, comp_occ_incl),
        },
        'revpar_excl_ooo': {
            'current': current_revpar_excl,
            'comparison': comp_revpar_excl,
            **_compute_delta(current_revpar_excl, comp_revpar_excl),
        },
        'revpar_incl_ooo': {
            'current': current_revpar_incl,
            'comparison': comp_revpar_incl,
            **_compute_delta(current_revpar_incl, comp_revpar_incl),
        },
    }


def _align_monthly_data(
    current_months: List[Dict],
    comparison_months: Optional[List[Dict]],
) -> List[Dict]:
    """Align current and comparison monthly data by month key.

    Returns list of dicts with current, comparison, delta, and pickup per month.
    """
    # Index comparison by month
    comp_by_month = {}  # type: Dict[str, Dict]
    if comparison_months:
        for m in comparison_months:
            comp_by_month[m['month']] = m

    result = []
    for cur in current_months:
        month_key = cur['month']
        comp = comp_by_month.get(month_key)

        entry = {
            'month': cur['month'],
            'label': cur['label'],
            'current': {
                'rooms': cur['rooms'],
                'revenue': _safe_float(cur['revenue']),
                'adr': cur['adr'],
                'occ_excl_ooo': cur['occ_excl_ooo'],
                'occ_incl_ooo': cur['occ_incl_ooo'],
                'revpar_excl_ooo': cur['revpar_excl_ooo'],
                'revpar_incl_ooo': cur['revpar_incl_ooo'],
                'available_room_nights': cur['available_room_nights'],
                'ooo_room_nights': cur['ooo_room_nights'],
            },
        }

        if comp:
            entry['comparison'] = {
                'rooms': comp['rooms'],
                'revenue': _safe_float(comp['revenue']),
                'adr': comp['adr'],
                'occ_excl_ooo': comp['occ_excl_ooo'],
                'occ_incl_ooo': comp['occ_incl_ooo'],
                'revpar_excl_ooo': comp['revpar_excl_ooo'],
                'revpar_incl_ooo': comp['revpar_incl_ooo'],
                'available_room_nights': comp['available_room_nights'],
                'ooo_room_nights': comp['ooo_room_nights'],
            }
            entry['delta'] = {
                'rooms': cur['rooms'] - comp['rooms'],
                'revenue': round(_safe_float(cur['revenue']) - _safe_float(comp['revenue']), 2) if cur['revenue'] is not None and comp['revenue'] is not None else None,
                'adr': round(cur['adr'] - comp['adr'], 2) if cur['adr'] is not None and comp['adr'] is not None else None,
                'occ_excl_ooo_pp': round(cur['occ_excl_ooo'] - comp['occ_excl_ooo'], 2) if cur['occ_excl_ooo'] is not None and comp['occ_excl_ooo'] is not None else None,
                'occ_incl_ooo_pp': round(cur['occ_incl_ooo'] - comp['occ_incl_ooo'], 2) if cur['occ_incl_ooo'] is not None and comp['occ_incl_ooo'] is not None else None,
                'revpar_excl_ooo': round(cur['revpar_excl_ooo'] - comp['revpar_excl_ooo'], 2) if cur['revpar_excl_ooo'] is not None and comp['revpar_excl_ooo'] is not None else None,
                'revpar_incl_ooo': round(cur['revpar_incl_ooo'] - comp['revpar_incl_ooo'], 2) if cur['revpar_incl_ooo'] is not None and comp['revpar_incl_ooo'] is not None else None,
            }
            entry['pickup'] = _compute_pickup(
                {'rooms': cur['rooms'], 'revenue': cur['revenue']},
                {'rooms': comp['rooms'], 'revenue': comp['revenue']},
            )
        else:
            entry['comparison'] = None
            entry['delta'] = None
            entry['pickup'] = {'rooms': None, 'revenue': None, 'adr': None}

        result.append(entry)

    return result


def build_mode_payload(
    current_months: List[Dict],
    comparison_months: Optional[List[Dict]],
    comparison_snapshot_date: Optional[date],
) -> Optional[Dict]:
    """Build the payload for one comparison mode.

    Returns None if comparison_snapshot_date is None (mode unavailable).
    """
    if comparison_snapshot_date is None:
        return None

    return {
        'comparison_snapshot_date': comparison_snapshot_date.isoformat(),
        'kpi_banner': _build_kpi_banner(current_months, comparison_months),
        'monthly': _align_monthly_data(current_months, comparison_months),
    }


def build_current_only_payload(current_months: List[Dict]) -> Dict:
    """Build a minimal payload when no comparison is available at all."""
    return {
        'comparison_snapshot_date': None,
        'kpi_banner': _build_kpi_banner(current_months, None),
        'monthly': _align_monthly_data(current_months, None),
    }
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline && python -m pytest tests/test_payload.py -v -m "not integration"`
Expected: 8 PASSED

- [ ] **Step 5: Commit**

```bash
cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline
git add src/dashboard/payload.py tests/test_payload.py
git commit -m "feat(dashboard): add payload builder with null handling and KPI aggregation"
```

---

## Task 4: Hero Pacing Chart — SVG Generator

Generate the combination chart (revenue bars + comparison ghost bars + ADR line) as an SVG string.

**Files:**
- Create: `pace-pipeline/src/dashboard/charts.py`
- Create: `pace-pipeline/tests/test_charts.py`

- [ ] **Step 1: Write failing tests**

```python
# tests/test_charts.py
from __future__ import annotations

import pytest


class TestHeroPacingChart:
    def test_returns_svg_string(self):
        from src.dashboard.charts import render_hero_chart
        monthly = [
            {'label': 'Apr 2026', 'current': {'revenue': 2000000, 'adr': 550.0},
             'comparison': {'revenue': 1800000}},
            {'label': 'May 2026', 'current': {'revenue': 2200000, 'adr': 520.0},
             'comparison': {'revenue': 2100000}},
        ]
        svg = render_hero_chart(monthly)
        assert svg.startswith('<svg')
        assert '</svg>' in svg

    def test_includes_bars_and_line(self):
        from src.dashboard.charts import render_hero_chart
        monthly = [
            {'label': 'Apr 2026', 'current': {'revenue': 2000000, 'adr': 550.0},
             'comparison': {'revenue': 1800000}},
        ]
        svg = render_hero_chart(monthly)
        assert 'rect' in svg  # bars
        assert 'line' in svg or 'path' in svg or 'circle' in svg  # ADR markers

    def test_null_comparison_skips_ghost_bar(self):
        from src.dashboard.charts import render_hero_chart
        monthly = [
            {'label': 'Apr 2026', 'current': {'revenue': 2000000, 'adr': 550.0},
             'comparison': None},
        ]
        svg = render_hero_chart(monthly)
        assert svg.startswith('<svg')
        # Should have current bar but no ghost bar
        assert '#0A84FF' in svg  # current bar color
        assert '#C0C0C0' not in svg  # no ghost bar

    def test_null_adr_skips_point(self):
        from src.dashboard.charts import render_hero_chart
        monthly = [
            {'label': 'Apr 2026', 'current': {'revenue': 2000000, 'adr': None},
             'comparison': {'revenue': 1800000}},
        ]
        svg = render_hero_chart(monthly)
        assert svg.startswith('<svg')

    def test_zero_revenue_no_bar(self):
        from src.dashboard.charts import render_hero_chart
        monthly = [
            {'label': 'Apr 2026', 'current': {'revenue': 0, 'adr': None},
             'comparison': {'revenue': 1800000}},
        ]
        svg = render_hero_chart(monthly)
        assert svg.startswith('<svg')
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline && python -m pytest tests/test_charts.py -v -m "not integration"`
Expected: FAIL — `ModuleNotFoundError`

- [ ] **Step 3: Implement charts.py**

```python
# src/dashboard/charts.py
from __future__ import annotations

from typing import Dict, List, Optional

# Chart dimensions
CHART_WIDTH = 1060
CHART_HEIGHT = 320
PADDING_LEFT = 80
PADDING_RIGHT = 80
PADDING_TOP = 40
PADDING_BOTTOM = 50
PLOT_WIDTH = CHART_WIDTH - PADDING_LEFT - PADDING_RIGHT
PLOT_HEIGHT = CHART_HEIGHT - PADDING_TOP - PADDING_BOTTOM

# Colors from design spec
COLOR_CURRENT = '#0A84FF'
COLOR_GHOST = '#C0C0C0'
COLOR_ADR_LINE = '#1A1A1A'
COLOR_GRID = '#E5E5E5'
COLOR_LABEL = '#666666'


def _fmt_revenue_axis(val: float) -> str:
    """Format revenue for Y-axis label (e.g., '$2.0M', '$500K')."""
    if val >= 1_000_000:
        return '${:.1f}M'.format(val / 1_000_000)
    if val >= 1_000:
        return '${:.0f}K'.format(val / 1_000)
    return '${:.0f}'.format(val)


def _fmt_adr_axis(val: float) -> str:
    """Format ADR for secondary Y-axis."""
    return '${:.0f}'.format(val)


def render_hero_chart(monthly: List[Dict]) -> str:
    """Generate hero pacing chart as SVG string.

    Args:
        monthly: List of month dicts from the payload with keys:
            label, current.revenue, current.adr, comparison.revenue
            (comparison may be None)

    Returns:
        Complete <svg> element as a string.
    """
    if not monthly:
        return '<svg width="{}" height="60"><text x="50%" y="30" text-anchor="middle" fill="#666">No data available</text></svg>'.format(CHART_WIDTH)

    # Extract data
    labels = []
    current_rev = []
    ghost_rev = []
    adr_points = []

    for m in monthly:
        labels.append(m['label'].split(' ')[0][:3])  # "Apr 2026" -> "Apr"
        cur = m.get('current') or {}
        comp = m.get('comparison')

        rev = cur.get('revenue') or 0
        current_rev.append(float(rev))

        if comp and comp.get('revenue') is not None:
            ghost_rev.append(float(comp['revenue']))
        else:
            ghost_rev.append(None)

        adr = cur.get('adr')
        adr_points.append(adr)

    n = len(monthly)

    # Compute Y-axis scales
    all_rev = [r for r in current_rev if r > 0]
    all_rev += [r for r in ghost_rev if r is not None and r > 0]
    max_rev = max(all_rev) * 1.15 if all_rev else 1_000_000
    min_rev = 0.0

    valid_adr = [a for a in adr_points if a is not None and a > 0]
    if valid_adr:
        min_adr = min(valid_adr) * 0.85
        max_adr = max(valid_adr) * 1.15
    else:
        min_adr = 0
        max_adr = 1000

    # Bar geometry
    group_width = PLOT_WIDTH / n
    bar_width = group_width * 0.3
    gap = group_width * 0.05

    def rev_y(val: float) -> float:
        if max_rev == min_rev:
            return PADDING_TOP + PLOT_HEIGHT
        return PADDING_TOP + PLOT_HEIGHT - (val - min_rev) / (max_rev - min_rev) * PLOT_HEIGHT

    def adr_y(val: float) -> float:
        if max_adr == min_adr:
            return PADDING_TOP + PLOT_HEIGHT / 2
        return PADDING_TOP + PLOT_HEIGHT - (val - min_adr) / (max_adr - min_adr) * PLOT_HEIGHT

    parts = []
    parts.append('<svg width="{}" height="{}" xmlns="http://www.w3.org/2000/svg" style="font-family:Arial,Helvetica,sans-serif">'.format(CHART_WIDTH, CHART_HEIGHT))

    # Background
    parts.append('<rect width="{}" height="{}" fill="white"/>'.format(CHART_WIDTH, CHART_HEIGHT))

    # Grid lines (4 horizontal)
    for i in range(5):
        y = PADDING_TOP + (PLOT_HEIGHT / 4) * i
        val = max_rev - (max_rev / 4) * i
        parts.append('<line x1="{}" y1="{}" x2="{}" y2="{}" stroke="{}" stroke-width="0.5"/>'.format(
            PADDING_LEFT, y, CHART_WIDTH - PADDING_RIGHT, y, COLOR_GRID))
        parts.append('<text x="{}" y="{}" text-anchor="end" fill="{}" font-size="11">{}</text>'.format(
            PADDING_LEFT - 8, y + 4, COLOR_LABEL, _fmt_revenue_axis(val)))

    # ADR axis labels (right side)
    for i in range(5):
        y = PADDING_TOP + (PLOT_HEIGHT / 4) * i
        val = max_adr - (max_adr - min_adr) / 4 * i
        parts.append('<text x="{}" y="{}" text-anchor="start" fill="{}" font-size="11">{}</text>'.format(
            CHART_WIDTH - PADDING_RIGHT + 8, y + 4, COLOR_LABEL, _fmt_adr_axis(val)))

    # Bars
    baseline_y = rev_y(0)
    for i in range(n):
        group_x = PADDING_LEFT + i * group_width

        # Ghost bar (comparison) — left position
        if ghost_rev[i] is not None and ghost_rev[i] > 0:
            gh = ghost_rev[i]
            bar_top = rev_y(gh)
            bar_h = baseline_y - bar_top
            x = group_x + gap
            parts.append('<rect x="{:.1f}" y="{:.1f}" width="{:.1f}" height="{:.1f}" fill="none" stroke="{}" stroke-width="1.5" rx="2"/>'.format(
                x, bar_top, bar_width, bar_h, COLOR_GHOST))

        # Current bar — right position
        if current_rev[i] > 0:
            bar_top = rev_y(current_rev[i])
            bar_h = baseline_y - bar_top
            x = group_x + gap + bar_width + gap
            parts.append('<rect x="{:.1f}" y="{:.1f}" width="{:.1f}" height="{:.1f}" fill="{}" rx="2"/>'.format(
                x, bar_top, bar_width, bar_h, COLOR_CURRENT))

        # X-axis label
        label_x = group_x + group_width / 2
        parts.append('<text x="{:.1f}" y="{}" text-anchor="middle" fill="{}" font-size="11">{}</text>'.format(
            label_x, CHART_HEIGHT - 10, COLOR_LABEL, labels[i]))

    # ADR line
    adr_line_points = []
    for i in range(n):
        if adr_points[i] is not None:
            x = PADDING_LEFT + i * group_width + group_width / 2
            y = adr_y(adr_points[i])
            adr_line_points.append((x, y, adr_points[i]))

    if len(adr_line_points) > 1:
        path_d = 'M {:.1f} {:.1f}'.format(adr_line_points[0][0], adr_line_points[0][1])
        for x, y, _ in adr_line_points[1:]:
            path_d += ' L {:.1f} {:.1f}'.format(x, y)
        parts.append('<path d="{}" fill="none" stroke="{}" stroke-width="2"/>'.format(path_d, COLOR_ADR_LINE))

    # ADR dots
    for x, y, val in adr_line_points:
        parts.append('<circle cx="{:.1f}" cy="{:.1f}" r="3.5" fill="{}" stroke="white" stroke-width="1.5"/>'.format(
            x, y, COLOR_ADR_LINE))

    parts.append('</svg>')
    return '\n'.join(parts)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline && python -m pytest tests/test_charts.py -v -m "not integration"`
Expected: 5 PASSED

- [ ] **Step 5: Commit**

```bash
cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline
git add src/dashboard/charts.py tests/test_charts.py
git commit -m "feat(dashboard): add hero pacing chart SVG generator"
```

---

## Task 5: Jinja2 Template — HTML/CSS/JS

Create the complete dashboard template. This is the largest task — it produces the self-contained HTML file with embedded CSS and JS.

**Files:**
- Create: `pace-pipeline/src/dashboard/template.html`

- [ ] **Step 1: Create the Jinja2 template**

Write `pace-pipeline/src/dashboard/template.html` — a complete HTML file that:

1. Receives `payload_json` (the stringified DATA object) and `hero_chart_svg` (pre-rendered SVG string) from the Python generator
2. Embeds the DATA as `<script>const DATA = {{ payload_json }};</script>`
3. Contains all CSS in a single `<style>` block using the design spec's CSS variables
4. Contains all JS inline for interactivity:
   - `activeMode` state (default: first available mode, prefer 'stly')
   - `activeOoo` state (default: per hotel config)
   - `activeChartFilter` state (default: 12)
   - `render()` — rebuilds KPI banner, monthly table from DATA. Uses `getActiveData()` which returns `DATA.modes[activeMode]` when available, or `DATA.current_only` when `default_mode` is null (all-modes-unavailable fallback).
   - `getActiveData()` — returns the active mode's data, or `DATA.current_only` if all modes are null. All render paths use this — never access `DATA.modes[activeMode]` directly.
   - `setCompare(mode)` — switches comparison mode, re-renders
   - `setOoo(mode)` — switches OOO toggle, re-renders
   - `setChartFilter(n)` — re-renders chart with n months
   - `toggleDay(month)` — expand/collapse daily breakdown
   - Null-mode handling: `getActiveData()` abstracts this; render functions check `data.monthly[i].comparison === null` for per-month nulls
   - Format helpers: `fmtMoney()`, `fmtPct()`, `fmtDelta()`
   - When `DATA.default_mode === null`: all comparison pills grayed out, banner shows "No comparison data available", deltas/pickup all show "—"

The template structure follows the design spec sections 1-8: Report Header → Browser Controls → KPI Banner → Hero Chart → Monthly Table → Daily Breakdown → Footnotes → Footer.

**Key implementation details:**

- The hero chart SVG is injected server-side via `{{ hero_chart_svg | safe }}` for the default 12-month view. The JS `setChartFilter()` function re-renders the chart client-side by reading `DATA.modes[activeMode].monthly` and calling a JS version of the chart renderer for filtered views.
- Print CSS: `@media print` hides controls, filter pills, daily rows; forces 12-month chart; preserves colors with `-webkit-print-color-adjust: exact`
- All delta cells use CSS classes `.pos` (green), `.neg` (red), `.neutral` (gray) applied by the JS render function based on value sign
- Daily breakdown rows use class `.day-expand-row` hidden by default, toggled by `toggleDay()`
- Null values render as `<span class="null">—</span>` via the `fmtMoney()` / `fmtPct()` helpers

This file will be approximately 400-600 lines. The full template content will be written by the implementing agent using the design spec as the source of truth for layout, colors, columns, and behavior.

- [ ] **Step 2: Commit**

```bash
cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline
git add src/dashboard/template.html
git commit -m "feat(dashboard): add Jinja2 template with full v5 layout"
```

---

## Task 6: Generator Script — CLI Entry Point

Ties everything together: query Supabase → find comparison snapshots → build payload → render template → write HTML.

**Files:**
- Create: `pace-pipeline/src/dashboard/generate.py`

- [ ] **Step 1: Write the generator**

```python
# src/dashboard/generate.py
from __future__ import annotations

import argparse
import json
import os
import sys
from datetime import date
from decimal import Decimal
from pathlib import Path
from typing import Dict, List, Optional

from jinja2 import Environment, FileSystemLoader

from src.dashboard.charts import render_hero_chart
from src.dashboard.payload import build_mode_payload, build_current_only_payload, _align_stly_date
from src.dashboard.snapshot_finder import find_comparison_snapshot
from src.db import get_client, get_hotel
from src.queries import monthly_aggregate_v2, rolling_12_month_window


class DecimalEncoder(json.JSONEncoder):
    """JSON encoder that converts Decimal to float."""
    def default(self, obj):
        if isinstance(obj, Decimal):
            return float(obj)
        return super().default(obj)


def generate_dashboard(
    hotel_id: str,
    snapshot_date: date,
    output_path: str,
) -> None:
    """Generate a v5 dashboard HTML file for a hotel.

    Args:
        hotel_id: Hotel code (e.g., 'KAH')
        snapshot_date: The snapshot date to generate from
        output_path: Where to write the HTML file
    """
    client = get_client()
    hotel = get_hotel(client, hotel_id)
    if not hotel:
        print('Error: Hotel {} not found'.format(hotel_id), file=sys.stderr)
        sys.exit(1)

    # Compute rolling 12-month window
    window_start, window_end = rolling_12_month_window(snapshot_date)

    # Get current monthly data
    all_months = monthly_aggregate_v2(hotel_id, snapshot_date)
    # Filter to rolling 12-month window
    current_months = [
        m for m in all_months
        if window_start.strftime('%Y-%m') <= m['month'] <= window_end.strftime('%Y-%m')
    ]

    # Find comparison snapshots and build mode payloads
    modes = {}  # type: Dict[str, Optional[Dict]]
    for mode in ('stly', 'wow', 'budget'):
        comp_date = find_comparison_snapshot(hotel_id, snapshot_date, mode)
        if comp_date is None:
            modes[mode] = None
            continue

        # Get comparison monthly data for the same stay-date window
        comp_all_months = monthly_aggregate_v2(hotel_id, comp_date)

        # Align comparison months to current year for STLY
        year_offset = snapshot_date.year - comp_date.year
        if year_offset != 0:
            for m in comp_all_months:
                # Shift month key forward by year_offset
                parts = m['month'].split('-')
                new_year = int(parts[0]) + year_offset
                m['month'] = '{}-{}'.format(new_year, parts[1])
                m['label'] = date(new_year, int(parts[1]), 1).strftime('%b %Y')

        comp_months = [
            m for m in comp_all_months
            if window_start.strftime('%Y-%m') <= m['month'] <= window_end.strftime('%Y-%m')
        ]

        modes[mode] = build_mode_payload(current_months, comp_months, comp_date)

    # Determine default active mode
    default_mode = None
    for preferred in ('stly', 'wow'):
        if modes.get(preferred) is not None:
            default_mode = preferred
            break

    # Build hero chart SVG (use default mode or current-only)
    if default_mode and modes[default_mode]:
        chart_monthly = modes[default_mode]['monthly']
    else:
        chart_monthly = [
            {'label': m['label'], 'current': {'revenue': float(m['revenue']), 'adr': m['adr']}, 'comparison': None}
            for m in current_months
        ]
    hero_svg = render_hero_chart(chart_monthly)

    # Build current-only payload (always available as fallback)
    current_only = build_current_only_payload(current_months)

    # Assemble full payload
    payload = {
        'hotel': {
            'id': hotel.id,
            'name': hotel.name,
            'total_rooms': hotel.total_rooms,
            'occ_excludes_ooo': getattr(hotel, 'occ_excludes_ooo', False),
        },
        'snapshot_date': snapshot_date.isoformat(),
        'window': {
            'start': window_start.isoformat(),
            'end': window_end.isoformat(),
            'months': 12,
        },
        'default_mode': default_mode,
        'modes': modes,
        'current_only': current_only,
    }

    payload_json = json.dumps(payload, cls=DecimalEncoder, indent=2)

    # Render template
    template_dir = os.path.dirname(os.path.abspath(__file__))
    env = Environment(loader=FileSystemLoader(template_dir))
    template = env.get_template('template.html')

    html = template.render(
        payload_json=payload_json,
        hero_chart_svg=hero_svg,
        hotel_name=hotel.name,
        snapshot_date=snapshot_date.isoformat(),
    )

    # Write output
    output = Path(output_path)
    output.parent.mkdir(parents=True, exist_ok=True)
    output.write_text(html, encoding='utf-8')
    print('Dashboard generated: {}'.format(output_path))


def main():
    # type: () -> None
    parser = argparse.ArgumentParser(description='Generate pace dashboard v5')
    parser.add_argument('hotel_id', help='Hotel code (e.g., KAH)')
    parser.add_argument('snapshot_date', help='Snapshot date (YYYY-MM-DD)')
    parser.add_argument('-o', '--output', default=None,
                        help='Output HTML file path (default: dashboard-{hotel_id}.html)')
    args = parser.parse_args()

    snap_date = date.fromisoformat(args.snapshot_date)
    output = args.output or 'dashboard-{}.html'.format(args.hotel_id.lower())

    generate_dashboard(args.hotel_id, snap_date, output)


if __name__ == '__main__':
    main()
```

- [ ] **Step 2: Commit**

```bash
cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline
git add src/dashboard/generate.py
git commit -m "feat(dashboard): add CLI generator script"
```

---

## Task 7: Integration Test — Generate for KAH

End-to-end test: generate a real dashboard for KAH using live Supabase data and verify the output.

**Files:**
- Create: `pace-pipeline/tests/test_dashboard_integration.py`

- [ ] **Step 1: Write integration test**

```python
# tests/test_dashboard_integration.py
from __future__ import annotations

import json
import os
import tempfile
from datetime import date
from pathlib import Path

import pytest

pytestmark = pytest.mark.integration


class TestDashboardGeneration:
    def test_generate_kah_dashboard(self):
        """Generate a dashboard for KAH and verify basic structure."""
        from src.dashboard.generate import generate_dashboard

        with tempfile.NamedTemporaryFile(suffix='.html', delete=False) as f:
            output_path = f.name

        try:
            # Use a known snapshot date for KAH
            generate_dashboard('KAH', date(2026, 3, 11), output_path)

            html = Path(output_path).read_text(encoding='utf-8')

            # Basic structure checks
            assert 'The Kahala Hotel & Resort' in html
            assert 'const DATA =' in html
            assert '<svg' in html  # hero chart
            assert '2026-03-11' in html  # snapshot date

            # Extract and validate JSON payload
            start = html.index('const DATA = ') + len('const DATA = ')
            end = html.index(';\n', start)
            payload = json.loads(html[start:end])

            assert payload['hotel']['id'] == 'KAH'
            assert payload['hotel']['total_rooms'] == 338
            assert payload['hotel']['occ_excludes_ooo'] is True
            assert payload['window']['months'] == 12

            # Should have STLY mode (KAH has data back to 2024)
            assert payload['modes']['stly'] is not None
            assert payload['modes']['stly']['comparison_snapshot_date'] is not None

            # Monthly data should have 12 months
            stly = payload['modes']['stly']
            assert len(stly['monthly']) >= 10  # might not have full 12 yet

            # KPI banner should have revenue
            assert stly['kpi_banner']['revenue']['current'] > 0

            # Budget should be null
            assert payload['modes']['budget'] is None

        finally:
            os.unlink(output_path)

    def test_generate_handles_missing_comparison(self):
        """Generate dashboard for a hotel with limited data."""
        from src.dashboard.generate import generate_dashboard

        with tempfile.NamedTemporaryFile(suffix='.html', delete=False) as f:
            output_path = f.name

        try:
            # LOT has very limited data — STLY likely missing
            generate_dashboard('LOT', date(2026, 3, 12), output_path)

            html = Path(output_path).read_text(encoding='utf-8')
            assert 'Lotus Honolulu' in html
            assert 'const DATA =' in html

            # Verify current_only payload always exists
            start = html.index('const DATA = ') + len('const DATA = ')
            end = html.index(';\n', start)
            payload = json.loads(html[start:end])

            # current_only must always be present
            assert 'current_only' in payload
            assert payload['current_only'] is not None
            assert 'kpi_banner' in payload['current_only']
            assert 'monthly' in payload['current_only']

            # If no comparison modes available, default_mode may be None
            if payload['default_mode'] is None:
                # Verify current_only has valid data
                assert len(payload['current_only']['monthly']) > 0

        finally:
            os.unlink(output_path)
```

- [ ] **Step 2: Run integration test**

Run: `cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline && python -m pytest tests/test_dashboard_integration.py -v`
Expected: 2 PASSED (requires `.env` with Supabase credentials)

- [ ] **Step 3: Generate a real dashboard and copy to preview repo**

```bash
cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline
python -m src.dashboard.generate KAH 2026-03-11 -o /Users/kinmengsio/Developer/lights-on/pace-dashboard-preview/dashboard-v5.html
```

- [ ] **Step 4: Open dashboard-v5.html in browser and verify**

Manual verification checklist:
- [ ] KPI banner shows 4 metrics with deltas
- [ ] Hero chart shows 12 monthly bars with ADR line
- [ ] Time filter pills (3/6/9/12) work — chart re-renders, table unchanged
- [ ] Comparison toggle (WoW/STLY) updates all sections
- [ ] OOO toggle updates occupancy and RevPAR values
- [ ] Budget pill is grayed out
- [ ] Monthly table has 12 rows with all columns
- [ ] Clicking a month row expands daily breakdown
- [ ] Pickup ADR shows "—" where pickup rooms = 0
- [ ] Print preview (Cmd+P) shows clean 2-page layout, no controls visible

- [ ] **Step 5: Commit all remaining files**

```bash
cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline
git add tests/test_dashboard_integration.py
git commit -m "test(dashboard): add integration tests for dashboard generation"

cd /Users/kinmengsio/Developer/lights-on/pace-dashboard-preview
git add dashboard-v5.html
git commit -m "feat: add generated v5 dashboard for KAH"
```

---

## Task 8: Daily Breakdown Data

Extend the payload to include per-day data for the daily breakdown expansion. This reuses the existing `daily_comparison` query with minor adaptation.

**Files:**
- Modify: `pace-pipeline/src/dashboard/generate.py`
- Modify: `pace-pipeline/src/dashboard/payload.py`

- [ ] **Step 1: Add daily data builder to payload.py**

Add to `pace-pipeline/src/dashboard/payload.py`:

```python
def build_daily_data(
    hotel_id: str,
    snapshot_date: date,
    comparison_date: Optional[date],
    months: List[str],
    snapshot_type: str = 'otb',
) -> Dict[str, List[Dict]]:
    """Build daily breakdown data for each month.

    Args:
        hotel_id: Hotel code
        snapshot_date: Current snapshot date
        comparison_date: Comparison snapshot date (None if unavailable)
        months: List of month keys ('2026-04', etc.)
        snapshot_type: Snapshot type

    Returns:
        Dict keyed by month ('2026-04' -> list of day dicts)
    """
    from src.queries import daily_comparison, daily_pickup, _fetch_snapshot_rows
    from src.db import get_client, get_hotel

    client = get_client()
    hotel = get_hotel(client, hotel_id)

    # Fetch all rows for current snapshot
    current_rows = _fetch_snapshot_rows(hotel_id, snapshot_date, snapshot_type)
    comp_rows = _fetch_snapshot_rows(hotel_id, comparison_date, snapshot_type) if comparison_date else []

    # Compute year offset for STLY alignment
    year_offset = (snapshot_date.year - comparison_date.year) if comparison_date else 0

    # Index rows by stay_date
    from datetime import date as date_type
    cur_by_date = {}  # type: Dict[str, Dict]
    for row in current_rows:
        cur_by_date[row['stay_date']] = row

    comp_by_date = {}  # type: Dict[str, Dict]
    for row in comp_rows:
        sd = row['stay_date']
        if year_offset != 0:
            d = date_type.fromisoformat(sd)
            aligned = _align_stly_date(d, year_offset)
            if aligned is None:
                continue  # skip leap-day rows that can't align
            sd = aligned.isoformat()
        comp_by_date[sd] = row

    result = {}  # type: Dict[str, List[Dict]]
    for month_key in months:
        days = []
        # Filter current rows to this month
        for sd_str, row in sorted(cur_by_date.items()):
            if not sd_str.startswith(month_key):
                continue

            rooms = int(row['rooms_sold'] or 0)
            # Use room_revenue consistently (same basis as monthly/KPI)
            raw_room_rev = row.get('room_revenue')
            revenue = float(raw_room_rev) if raw_room_rev is not None else float(row['total_revenue'] or 0)
            adr = round(revenue / rooms, 2) if rooms > 0 else None

            available = int(row.get('available_rooms') or 0)
            ooo = int(row.get('ooo_rooms') or 0)
            # Denominators match spec and monthly_aggregate_v2:
            #   excl OOO = (rooms_sold + available_rooms) = total - ooo
            #   incl OOO = (rooms_sold + available_rooms + ooo_rooms) = total
            denom_excl = rooms + available
            denom_incl = rooms + available + ooo

            occ_excl = round(rooms / denom_excl * 100, 2) if denom_excl > 0 else None
            occ_incl = round(rooms / denom_incl * 100, 2) if denom_incl > 0 else None
            revpar_excl = round(revenue / denom_excl, 2) if denom_excl > 0 else None
            revpar_incl = round(revenue / denom_incl, 2) if denom_incl > 0 else None

            day = {
                'date': sd_str,
                'current': {
                    'rooms': rooms,
                    'revenue': revenue,
                    'adr': adr,
                    'occ_excl_ooo': occ_excl,
                    'occ_incl_ooo': occ_incl,
                    'revpar_excl_ooo': revpar_excl,
                    'revpar_incl_ooo': revpar_incl,
                },
            }

            # Comparison data
            comp = comp_by_date.get(sd_str)
            if comp:
                c_rooms = int(comp['rooms_sold'] or 0)
                # Use room_revenue consistently for comparison too
                c_raw_room_rev = comp.get('room_revenue')
                c_revenue = float(c_raw_room_rev) if c_raw_room_rev is not None else float(comp['total_revenue'] or 0)
                c_adr = round(c_revenue / c_rooms, 2) if c_rooms > 0 else None

                pickup_rooms = rooms - c_rooms
                pickup_revenue = revenue - c_revenue
                pickup_adr = round(pickup_revenue / pickup_rooms, 2) if pickup_rooms != 0 else None

                day['comparison'] = {
                    'rooms': c_rooms,
                    'revenue': c_revenue,
                    'adr': c_adr,
                }
                day['delta'] = {
                    'rooms': rooms - c_rooms,
                    'revenue': round(revenue - c_revenue, 2),
                    'adr': round(adr - c_adr, 2) if adr is not None and c_adr is not None else None,
                }
                day['pickup'] = {
                    'rooms': pickup_rooms,
                    'revenue': round(pickup_revenue, 2),
                    'adr': pickup_adr,
                }
            else:
                day['comparison'] = None
                day['delta'] = None
                day['pickup'] = {'rooms': None, 'revenue': None, 'adr': None}

            days.append(day)

        result[month_key] = days

    return result
```

- [ ] **Step 2: Update generate.py to include daily data**

In `generate_dashboard()`, after building modes, add daily data for each mode:

```python
    # Add daily data to each mode
    for mode_key, mode_payload in modes.items():
        if mode_payload is None:
            continue
        comp_date = date.fromisoformat(mode_payload['comparison_snapshot_date'])
        month_keys = [m['month'] for m in current_months]
        daily = build_daily_data(hotel_id, snapshot_date, comp_date, month_keys)
        mode_payload['daily'] = daily
```

Add `build_daily_data` to the imports from `src.dashboard.payload`.

- [ ] **Step 3: Commit**

```bash
cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline
git add src/dashboard/payload.py src/dashboard/generate.py
git commit -m "feat(dashboard): add daily breakdown data to payload"
```

---

## Task 9: Visual Mockup Review

Generate the final dashboard with all features, open in browser, and do a visual review against the design spec.

**Files:**
- No new files — this is a verification task

- [ ] **Step 1: Regenerate dashboard with daily data**

```bash
cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline
python -m src.dashboard.generate KAH 2026-03-11 -o /Users/kinmengsio/Developer/lights-on/pace-dashboard-preview/dashboard-v5.html
```

- [ ] **Step 2: Run daily-to-monthly reconciliation check**

After generating the KAH dashboard, extract the payload and verify that daily rows sum to monthly totals:

```python
# Inline verification (run in Python REPL or as a script)
import json
from pathlib import Path

html = Path('/Users/kinmengsio/Developer/lights-on/pace-dashboard-preview/dashboard-v5.html').read_text()
start = html.index('const DATA = ') + len('const DATA = ')
end = html.index(';\n', start)
payload = json.loads(html[start:end])

mode = payload['modes'].get('stly') or payload['current_only']
for month_data in mode['monthly']:
    month_key = month_data['month']
    daily = mode.get('daily', {}).get(month_key, [])
    if not daily:
        continue
    daily_rooms = sum(d['current']['rooms'] for d in daily)
    daily_revenue = sum(d['current']['revenue'] for d in daily)
    assert daily_rooms == month_data['current']['rooms'], \
        f"{month_key}: daily rooms {daily_rooms} != monthly {month_data['current']['rooms']}"
    assert abs(daily_revenue - month_data['current']['revenue']) < 0.02, \
        f"{month_key}: daily revenue {daily_revenue} != monthly {month_data['current']['revenue']}"
    print(f"{month_key}: OK (rooms={daily_rooms}, revenue=${daily_revenue:,.2f})")
```

- [ ] **Step 3: Open in browser and verify all acceptance criteria**

Verification against design spec acceptance criteria:

**KPI Banner:**
- [ ] Shows rolling 12-month aggregates with deltas
- [ ] OOO toggle updates Occupancy and RevPAR only

**Hero Chart:**
- [ ] 12 monthly bar groups by default
- [ ] Time filter pills (3/6/9/12) re-render chart only
- [ ] Comparison toggle updates ghost bars
- [ ] Print shows 12 months, no pills

**Monthly Breakdown:**
- [ ] 12 rows with all columns including Pickup ADR
- [ ] Click expands daily breakdown
- [ ] Pickup ADR = "—" when pickup rooms = 0

**Comparison Modes:**
- [ ] Budget pill grayed out
- [ ] WoW ↔ STLY toggle updates everything

**Print:**
- [ ] Cmd+P shows clean layout
- [ ] OOO selection honored in print
- [ ] Controls hidden, daily rows hidden

- [ ] **Step 3: Fix any visual issues found**

Address any layout, spacing, or data issues discovered during review.

- [ ] **Step 4: Final commit**

```bash
cd /Users/kinmengsio/Developer/lights-on/pace-dashboard-preview
git add dashboard-v5.html
git commit -m "fix: visual polish after review"
```

---

## Run All Tests

After all tasks complete:

```bash
# Unit tests (no DB needed)
cd /Users/kinmengsio/Developer/lights-on/revenue-reporting-automation/pace-pipeline
python -m pytest tests/test_dashboard_queries.py tests/test_snapshot_finder.py tests/test_payload.py tests/test_charts.py -v -m "not integration"

# Integration tests (needs .env)
python -m pytest tests/test_dashboard_integration.py -v
```

Expected: All tests pass. Dashboard generates in <5 seconds. HTML output matches design spec.
