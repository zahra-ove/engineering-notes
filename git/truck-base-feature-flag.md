# Trunk-Based Development & Feature Flags

### Trunk-Based Development

A development approach where developers make **small, incremental changes** and merge them into the main branch frequently.

Typical workflow:

```text
Small Change
     ↓
Short-lived Branch
     ↓
Pull Request
     ↓
main
     ↓
CI / Tests
     ↓
Staging
```

The goal is to avoid long-lived feature branches and keep `main` continuously deployable.

### Feature Flags

A **Feature Flag** is a switch that controls whether a feature is enabled or disabled, independently of the code deployment.

```go
if featureFlag("new-login") {
    newLogin()
} else {
    oldLogin()
}
```

For example:

```text
new-login = OFF
→ New code is deployed, but users still use the old login.

new-login = ON
→ The new login becomes active.
```

### Key Idea

**Deployment ≠ Release**

A feature can be deployed to `main` and staging while remaining disabled through a Feature Flag.

This allows the team to:

* Merge small changes frequently.
* Deploy changes early.
* Test features in staging before releasing them.
* Reduce long-lived branches and merge conflicts.
* Gradually enable new features when they are ready.
