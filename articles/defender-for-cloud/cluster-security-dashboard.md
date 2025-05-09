---
title: Azure Kubernetes Service (AKS) Security Dashboard
description: Explain the views and actions in the AKS security Dashboard to enable and protect an individual cluster with Defender for Containers.
ms.author: dacurwin
ms.topic: conceptual
ms.date: 12/19/2024

CustomerIntent: As the person responsible for the security of a cluster, I want know what views and actions the cluster security dashboard provides, so I can remediate security issues in the cluster.
---

# Azure Kubernetes Service (AKS) Security Dashboard (Preview)

The AKS security dashboard provides comprehensive visibility and automated remediation capabilities for security issues, empowering platform engineering teams to easily, and more effectively, secure their Kubernetes environment.

Consolidating security and operational data in one place directly within the AKS portal allows engineers to benefit from a unified view of their Kubernetes environment. The view enables more efficient detection and remediation of security issues, with minimal disruption to their workflows to reduce the risk of overlooking security issues and improving remediation cycles.

The AKS Security Dashboard allows the user to:

- View the security posture of the cluster.
- Enable Defender for Containers plan and configure settings for the specific cluster resource.
- Review vulnerability assessment recommendations.
- Review configurations of the cluster and running containers that deviate from security best practices ("misconfigurations"), together with guided or automated remediation.
- Assign an owner to a recommendation or misconfiguration responsible for remediation by a specified date. (Available with Defender Cloud Security Posture Management (DCSPM) enabled for the subscription.)

## Prerequisites

The AKS Security Dashboard shows security vulnerabilities and misconfigurations for a cluster if at least one of the following plans is enabled:

- [Defender for Containers on the subscription](tutorial-enable-containers-azure.md) or [the individual cluster](#setting-the-defender-for-containers-plan)
- [DCSPM](tutorial-enable-cspm-plan.md) on the subscription

## Using the AKS Security Dashboard

The AKS Security Dashboard is accessed from a cluster resource pane by selecting **Microsoft Defender for Cloud** in the menu list. The dashboard shows:

- The **Security findings for this cluster** summary. It shows the total vulnerabilities and misconfigurations by risk level.
- The **Vulnerabilities** and **Misconfigurations** tabs. Each of the tabs shows a summary of the risk level and list of recommendations.
- The **Microsoft Defender for Containers status** for the cluster and the option to configure its coverage.

### Remediation of vulnerabilities and misconfigurations

In both the **Vulnerabilities** and **Misconfiguration** tabs, selecting one of the recommendations opens a pane showing the full details of the recommendation. In the details pane, the user can:

- View the full details of the recommendation and the steps for remediation.
- Remediate the issue by selecting the **Quick Fix** option.
- Assign an owner that is responsible for remediation of the issue. Selecting **Assign Owner** opens a pane where the owner name, remediation time frame, periodic email reminders can be set.

The user can select multiple recommendations using the checkbox beside each recommendation and then select **Assign owner** in the dashboard ruler, to assign a single owner to all of them.

Selecting **Download CSV report** downloads the cluster vulnerabilities and misconfigurations as a CSV file. The cluster vulnerabilities and misconfigurations can also be retrieved using the [Defender for Cloud REST API](/rest/api/defenderforcloud-composite/assessments).

### Setting the Defender for Containers plan

Selecting **Settings** of **Microsoft Defender for Containers status** opens a pane for the user to configure the Defender for Containers plan **for the specific cluster**. In the case where Defender for Containers is enabled on the subscription level, plan settings can only be changed on the subscription level. Plan configurations include the following settings:

- **Kubernetes API access** - Agentless Container Security Posture Management, runtime vulnerability assessment and response actions.
- **Registry access** - Agentless vulnerability assessment for registry images.
- **Azure policy** - Deployment of an agent on the cluster to generate recommendations for hardening the cluster control and data planes.

The Defender for Containers plan for the cluster can also be set using the [REST API commands](/rest/api/defenderforcloud-composite/pricings/update).
