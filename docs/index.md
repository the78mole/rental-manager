---
layout: home

hero:
  name: "Rental Manager"
  text: "Property Management Platform"
  tagline: Manage buildings, tenants, contracts and utility billing – open source, self-hosted, role-based.
  actions:
    - theme: brand
      text: Get Started
      link: /setup
    - theme: alt
      text: Architecture
      link: /architecture
    - theme: alt
      text: View on GitHub
      link: https://github.com/the78mole/rental-manager

features:
  - icon: 🏢
    title: Multi-Landlord
    details: Each landlord manages their own portfolio of buildings, apartments and tenants in full isolation.
  - icon: 🔐
    title: Keycloak OIDC
    details: Federated authentication via Keycloak with fine-grained role-based access control.
  - icon: 📊
    title: Utility Billing
    details: Automated Celery-based billing calculations from meter readings with PDF export.
  - icon: 🏗️
    title: Caretaker Model
    details: Hausverwalter can be assigned at building or apartment level with scoped access.
  - icon: 🐳
    title: Fully Containerised
    details: One-command setup with Docker Compose – FastAPI, React, Keycloak, PostgreSQL, Redis, RustFS.
  - icon: 📖
    title: Open Source
    details: MIT licensed. Dev Containers support for instant onboarding in VS Code and GitHub Codespaces.
---
