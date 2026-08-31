"""
FTTx/GPON Network Design Calculator
------------------------------------
A tool that automates two core tasks in FTTH/GPON network planning:

1. Optical Link Budget Calculation
   Determines whether a proposed fiber path (OLT -> splitters -> ONT at the
   home) stays within the optical loss budget of a chosen GPON class.

2. Bill of Quantities (BOQ) & Cost Estimation
   Calculates material quantities and total rollout cost for a given
   coverage area, based on homes passed, cable lengths, and splitter plan.

Author: <Akshay Salkar>
Purpose: Independent portfolio project demonstrating FTTx/GPON network
         design fundamentals combined with basic software engineering.
"""

from dataclasses import dataclass, field
from typing import List
import json


# ---------------------------------------------------------------------------
# Reference data (industry-standard typical values, in dB / per-unit cost)
# These can be tuned per-vendor or per-region.
# ---------------------------------------------------------------------------

# ITU-T G.984.2 GPON optical budget classes (approximate, in dB)
GPON_CLASSES = {
    "B+": 28.0,
    "C+": 32.0,
    "C++": 35.0,
}

# Typical splitter insertion loss (dB), including 1-2 dB safety margin
SPLITTER_LOSS_DB = {
    2: 3.7,
    4: 7.2,
    8: 10.5,
    16: 13.8,
    32: 17.5,
    64: 21.0,
}

FIBER_ATTENUATION_DB_PER_KM = 0.35   # singlemode fiber @ 1310/1490nm
CONNECTOR_LOSS_DB = 0.5              # per connector pair
SPLICE_LOSS_DB = 0.1                 # per fusion splice

# Illustrative unit costs in INR - replace with real vendor quotes for
# a specific country/market before using this for actual budgeting.
UNIT_COSTS_INR = {
    "feeder_fiber_per_km": 45000,       # armored feeder cable, per km, incl. duct
    "distribution_fiber_per_km": 32000,  # distribution cable, per km
    "drop_cable_per_home": 1200,         # drop cable + installation per home
    "splitter_1x8": 2500,
    "splitter_1x4": 1500,
    "splice_closure": 3500,
    "ont_per_home": 2200,
    "olt_port_amortized_per_home": 800,  # OLT port cost spread across homes on that port
    "labor_per_home": 1800,
}


@dataclass
class LinkBudgetResult:
    fiber_loss_db: float
    splitter_loss_db: float
    connector_loss_db: float
    splice_loss_db: float
    total_loss_db: float
    budget_db: float
    margin_db: float
    feasible: bool


@dataclass
class BOQResult:
    feeder_length_km: float
    distribution_length_km: float
    total_drop_cable_homes: int
    splitters_used: dict
    splice_closures: int
    total_cost_inr: float
    cost_per_home_inr: float
    cost_breakdown: dict


def calculate_link_budget(
    feeder_length_km: float,
    distribution_length_km: float,
    drop_length_km: float,
    splitter_stages: List[int],
    connector_count: int = 4,
    splice_count: int = 6,
    gpon_class: str = "C+",
) -> LinkBudgetResult:
    """
    Calculate the total optical loss from OLT to the furthest ONT and
    compare it against the chosen GPON class budget.

    splitter_stages: e.g. [4, 8] means a 1:4 splitter at the cabinet
                      followed by a 1:8 splitter near the home (1:32 total).
    """
    total_fiber_km = feeder_length_km + distribution_length_km + drop_length_km
    fiber_loss = total_fiber_km * FIBER_ATTENUATION_DB_PER_KM

    splitter_loss = 0.0
    for stage in splitter_stages:
        if stage not in SPLITTER_LOSS_DB:
            raise ValueError(f"Unsupported splitter ratio 1:{stage}")
        splitter_loss += SPLITTER_LOSS_DB[stage]

    connector_loss = connector_count * CONNECTOR_LOSS_DB
    splice_loss = splice_count * SPLICE_LOSS_DB

    total_loss = fiber_loss + splitter_loss + connector_loss + splice_loss
    budget = GPON_CLASSES[gpon_class]
    margin = budget - total_loss

    return LinkBudgetResult(
        fiber_loss_db=round(fiber_loss, 2),
        splitter_loss_db=round(splitter_loss, 2),
        connector_loss_db=round(connector_loss, 2),
        splice_loss_db=round(splice_loss, 2),
        total_loss_db=round(total_loss, 2),
        budget_db=budget,
        margin_db=round(margin, 2),
        feasible=margin >= 3.0,  # keep at least 3 dB safety margin
    )


def calculate_boq(
    homes_passed: int,
    feeder_length_km: float,
    distribution_length_km: float,
    splitter_stages: List[int],
) -> BOQResult:
    """
    Estimate materials and cost for the designed network segment.
    Assumes one splitter chain per ~homes_passed group; scale externally
    for larger areas by summing multiple BOQResult objects.
    """
    drop_cable_km = homes_passed * 0.05  # avg 50m drop per home, illustrative
    total_drop_cost = homes_passed * UNIT_COSTS_INR["drop_cable_per_home"]

    feeder_cost = feeder_length_km * UNIT_COSTS_INR["feeder_fiber_per_km"]
    distribution_cost = distribution_length_km * UNIT_COSTS_INR["distribution_fiber_per_km"]

    splitters_used = {}
    splitter_cost = 0.0
    for stage in splitter_stages:
        key = f"1x{stage}"
        splitters_used[key] = splitters_used.get(key, 0) + 1
        if stage <= 4:
            splitter_cost += UNIT_COSTS_INR["splitter_1x4"]
        else:
            splitter_cost += UNIT_COSTS_INR["splitter_1x8"]

    splice_closures = max(2, homes_passed // 50)  # 1 closure per ~50 homes
    splice_cost = splice_closures * UNIT_COSTS_INR["splice_closure"]

    ont_cost = homes_passed * UNIT_COSTS_INR["ont_per_home"]
    olt_cost = homes_passed * UNIT_COSTS_INR["olt_port_amortized_per_home"]
    labor_cost = homes_passed * UNIT_COSTS_INR["labor_per_home"]

    breakdown = {
        "feeder_fiber": round(feeder_cost, 2),
        "distribution_fiber": round(distribution_cost, 2),
        "drop_cable": round(total_drop_cost, 2),
        "splitters": round(splitter_cost, 2),
        "splice_closures": round(splice_cost, 2),
        "ont_equipment": round(ont_cost, 2),
        "olt_port_share": round(olt_cost, 2),
        "labor": round(labor_cost, 2),
    }
    total_cost = sum(breakdown.values())

    return BOQResult(
        feeder_length_km=feeder_length_km,
        distribution_length_km=distribution_length_km,
        total_drop_cable_homes=homes_passed,
        splitters_used=splitters_used,
        splice_closures=splice_closures,
        total_cost_inr=round(total_cost, 2),
        cost_per_home_inr=round(total_cost / homes_passed, 2),
        cost_breakdown=breakdown,
    )


def print_report(link_budget: LinkBudgetResult, boq: BOQResult, area_name: str):
    print("=" * 60)
    print(f"FTTH/GPON NETWORK DESIGN REPORT — {area_name}")
    print("=" * 60)

    print("\n--- Optical Link Budget ---")
    print(f"Fiber loss        : {link_budget.fiber_loss_db} dB")
    print(f"Splitter loss     : {link_budget.splitter_loss_db} dB")
    print(f"Connector loss    : {link_budget.connector_loss_db} dB")
    print(f"Splice loss       : {link_budget.splice_loss_db} dB")
    print(f"TOTAL LOSS        : {link_budget.total_loss_db} dB")
    print(f"GPON budget       : {link_budget.budget_db} dB")
    print(f"Margin            : {link_budget.margin_db} dB")
    print(f"Feasible?         : {'YES' if link_budget.feasible else 'NO - redesign needed'}")

    print("\n--- Bill of Quantities & Cost ---")
    print(f"Feeder fiber      : {boq.feeder_length_km} km")
    print(f"Distribution fiber: {boq.distribution_length_km} km")
    print(f"Homes passed      : {boq.total_drop_cable_homes}")
    print(f"Splitters used    : {boq.splitters_used}")
    print(f"Splice closures   : {boq.splice_closures}")
    print("\nCost breakdown (INR):")
    for item, cost in boq.cost_breakdown.items():
        print(f"  {item:<20}: Rs. {cost:,.2f}")
    print(f"\nTOTAL COST        : Rs. {boq.total_cost_inr:,.2f}")
    print(f"COST PER HOME     : Rs. {boq.cost_per_home_inr:,.2f}")
    print("=" * 60)


def export_json(link_budget: LinkBudgetResult, boq: BOQResult, path: str):
    data = {
        "link_budget": link_budget.__dict__,
        "boq": boq.__dict__,
    }
    with open(path, "w") as f:
        json.dump(data, f, indent=2)
    print(f"\nReport exported to {path}")


if __name__ == "__main__":
    # Example: a sample coverage area with 200 homes passed
    AREA_NAME = "Sample Area - 200 Homes Passed"
    HOMES_PASSED = 200
    FEEDER_KM = 2.5
    DISTRIBUTION_KM = 1.2
    DROP_KM = 0.05  # per home average, used inside link budget as total path
    SPLITTER_STAGES = [4, 8]  # 1:4 at cabinet, 1:8 at pole = 1:32 total split

    lb = calculate_link_budget(
        feeder_length_km=FEEDER_KM,
        distribution_length_km=DISTRIBUTION_KM,
        drop_length_km=DROP_KM,
        splitter_stages=SPLITTER_STAGES,
        connector_count=4,
        splice_count=6,
        gpon_class="C+",
    )

    boq = calculate_boq(
        homes_passed=HOMES_PASSED,
        feeder_length_km=FEEDER_KM,
        distribution_length_km=DISTRIBUTION_KM,
        splitter_stages=SPLITTER_STAGES,
    )

    print_report(lb, boq, AREA_NAME)
    export_json(lb, boq, "network_design_report.json")
