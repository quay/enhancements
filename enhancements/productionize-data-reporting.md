---
title: Quota Management and Enforcement on Quay
authors:
  - "@kwestpha"
reviewers:
  - "@kwestpha"
approvers:
  - "@kwestpha"
creation-date: 2022-09-29
last-updated: 2022-09-29
status: implementable
---

# Data Gathering and Reporting

Prototypes for analyzing Quay usage and reporting has been developed and now needs to be evaluated for value and then implemented into a full supported production product. 

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions

 > 1. What should we do with this data? 
 > 2. What cadence should this data be produced?
 > 3. How long should this data be kept?

## Summary

We have developed a prototype of Quay usage data from our CDN and other data systems. We need to evaluate the ROI of moving this to a more productionalized environment, followed by planning the project to do so. 

The project to productionalize this will include: 
* Terrafom project to create the spark cluster
* Designing the reports to output
* Building spark step execution automation
* Building PySpark jobs to execute and output the reports. 
* Develop ways to display this data in an easy to access manner. 

## Motivation

The motivation behind this project is to develop a way to produce Quay internal reporting. This could perhaps add a feature to Quay allowing customers to see other information and better understand the amount of usage the Quay instance is getting. 

### Goals

* Determine ROI of the data being produced. 
* Determine if the systems need to be productionalized.
* Determine where and how the reports need to be sent and used.  
* Plan the new systems 
* Develop the new systems to support goals above. 

### Non-Goals

* None at this time. 

## Design Details

* TBD

### Constraints

TBD

### Risks and Mitigations

* TBD

### Test Plan

* TBD

### Implementation History

* Prototype developed
