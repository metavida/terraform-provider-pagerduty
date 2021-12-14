---
layout: "pagerduty"
page_title: "PagerDuty: event_orchestration_separate_resources_per_type"
sidebar_current: "docs-pagerduty-resource-event-orchestration-router"
description: |-
  Creates and manages an Router associated with a Global Event Orchestration in PagerDuty.
---

# Separate resources for each "type" of Orchestration path

In this example, we have separate resources for each "type" of Orchestration:

* `pagerduty_event_orchestration_router` + `pagerduty_event_orchestration_router_rule`
* `pagerduty_event_orchestration_unrouted` + `pagerduty_event_orchestration_unrouted_rule`
* `pagerduty_event_orchestration_service` + `pagerduty_event_orchestration_service_rule`

## Pros

* We have the option of providing different arguments per-type.
* We have the option of documenting each type & which arguments it supports separately.

## Cons

* More types of resources & that the user has to remember & keep separate.

## Example of configuring a Global Event Orchestration

```hcl
resource "pagerduty_team" "engineering" {
  name = "Engineering"
}

resource "pagerduty_event_orchestration" "my_monitor" {
  name = "My Monitoring Orchestration"
  description = "Send events to a pair of services"
  team {
    id = pagerduty_team.foo.id
  }
}
```

## Example of configuring Router rules for an Orchestration

```hcl
# In the real world this resource may be defined "next to"
# the terraform code that configures the database service's monitors.
resource "pagerduty_event_orchestration_router_rule" "my_monitor_db" {
  name = "Events relating to our relational database"
  conditions = [
    { expression = "event.summary matches part 'database'" },
    { expression = "event.source matches regex 'db[0-9]+-server'" }
  ]
  route_to = pagerduty_service.database.id
}

# In the real world this resource may be defined "next to"
# the terraform code that configures the www service's monitors.
resource "pagerduty_event_orchestration_router_rule" "my_monitor_www" {
  name = "Events relating to our www app server"
  conditions = [
    { expression = "event.summary matches part 'www'" }
  ]
  route_to = pagerduty_service.www.id
}

resource "pagerduty_event_orchestration_router" "my_monitor" {
  event_orchestration = pagerduty_event_orchestration.my_monitor
  # Note: the router currently only supports a single set of rules.
  # If we change routers to support multiple sets in the future we'd need to introde a `sets` argument.
  rules = [
    pagerduty_event_orchestration_router_rule.my_monitor_db,
    pagerduty_event_orchestration_router_rule.my_monitor_www,
  ]
  catch_all = {
    route_to = "unrouted"
  }
}
```

## Example of configuring a Unrouted Rules for an Orchestration

```hcl
resource "pagerduty_event_orchestration_unrouted_rule" "critical" {
  name = "Update the summary of un-matched Critical alerts so they're easier to spot"
  conditions = [
    { expression = "event.severity matches 'critical'" }
  ]
  actions {
    extractions = [
      {
        target = "event.summary"
        template = "[Critical Unrouted] {{event.summary}}"
      }
    ]
  }
}

resource "pagerduty_event_orchestration_unrouted_rule" "match_everything" {
  name = "Reduce the severity of all other unrouted events"
  conditions = []
  actions = {
    severity = "info"
  }
}

resource "pagerduty_event_orchestration_unrouted" "my_monitor" {
  event_orchestration = pagerduty_event_orchestration.my_monitor
  sets = [
    {
      id = "start"
      rules = [
        pagerduty_event_orchestration_unrouted_rule.critical,
        pagerduty_event_orchestration_unrouted_rule.match_everything,
      ]
    }
  ]
  # In this example the user has defined their own "match_everything" rule with no conditions
  # so they aren't defining global catch_all behavior for the pagerduty_event_orchestration_unrouted_rule resource
  # but using the catch_all would be a totally legit alternative.
}
```

## Example of configuring a Service Orchestration

```hcl
resource "pagerduty_event_orchestration_service_rule", "start_all_events" {
  name = "Always apply some consistent event transformations to all events"
  conditions = []
  actions {
    variables {
      name = "hostname"
      path = "event.component"
      value = "hostname: (.*)"
      type = "regex"
    }
    extractions = [
      {
        # Demonstrating a template-style extraction
        template = "{{variables.hostname}}"
        target = "event.custom_details.hostname"
      },
      {
        # Demonstrating a regex-style extractions
        source = "event.source"
        regex = "www (.*) service"
        target = "event.source"
      }
    ]
    route_to = "step-two"                 
  }
}

resource "pagerduty_event_orchestration_service_rule", "step_two_critical" {
  name = "All critical alerts should be treated as P1 incidents"
  conditions = [
    { expression = "event.severity matches 'critical'" }
  ]
  actions {
    annotate = "Please use our P1 runbook: https://docs.test/p1-runbook"
    priority = "P0IN2KQ"
    suppress = false
  }
}

resource "pagerduty_event_orchestration_service_rule", "step_two_canary" {
  name = "If there's something wrong on the canary let the team know about it in our deployments Slack channel"
  conditions = [
    { expression = "event.custom_details.hostname matches part 'canary'" }
  ]
  actions {
    automation_actions = [
      {
        name = "Canary Slack Notification"
        url = "https://our-slack-listerner.test/canary-notification"
        auto_send = true
        parameters = []
      }
    ]
  }
}

resource "pagerduty_event_orchestration_service_rule", "step_two_info_off_hours" {
  name = "Never bother the on-call for info-level events outside of work hours"
  conditions = [
    { expression = "event.severity matches 'info' and not (now in Mon,Tue,Wed,Thu,Fri 09:00:00 to 17:00:00 America/Los_Angeles)" }
  ]
  actions {
    suppress = true
  }
}

resource "pagerduty_event_orchestration_service" "www" {
  service = route_to = pagerduty_service.www.id
  sets = [
    {
      id: "start"
      rules = [
        pagerduty_event_orchestration_service_rule.start_all_events
      ]
    },
    {
      id: "step-two"
      rules = [
        pagerduty_event_orchestration_service_rule.step_two_critical,
        pagerduty_event_orchestration_service_rule.step_two_canary,
        pagerduty_event_orchestration_service_rule.step_two_info_off_hours
      ]
    }
  ]
  catch_all = {
    suppress = true
  }
}
```

## Argument Reference

TBD