# Ozone Bootcamp
## Storage Container Manager
21 Jan 2025

### Start-up
#### What is Primordial SCM?
#### init vs bootstrap
#### What happens in init/bootstrap?
#### What happens in SCM start?
#### Exercise
- Start SCM Without init/bootstrap
- Run init on non-primordial SCM
- Run bootstrap on primordial SCM
- Run bootstrap on non-primordial SCM without primordial SCM running
- Run init on primordial SCM
- Run bootstrap on non-primordial SCM (with primordial SCM running)
- Re-run init on primordial SCM
- Re-run bootstrap on non-primordial SCM

### Safemode
Safemode is a state where the Ozone cluster operates with certain restrictions.

#### Safemode Rules
- Datanode Safemode Rule
- Healthy Pipeline Safemode Rule
- Container Safemode Rule
- One Replica Pipeline Safemode Rule
#### Safemode Behaviour
##### OM Operations
| Client Operation | Expected Behaviour |
|------------------|--------------------|
| Volume Create    | Should Succeed.    |
| Bucket Create    | Should Succeed.    |
| Volume List      | Should Succeed.    |
| Bucket List      | Should Succeed.    |
| Key/File List    | Should Succeed.    |
| Key/File Write   | Should fail with Safemode Exception |
| Key/File Read    | It should succeed if at least one DN holding the replica is registered. |

##### SCM Admin Operations
| Admin Command    |Expected Behaviour |
|------------------|-------------------|
|Close Container   | Should Succeed.   |
|CreatePipeline    | Should fail with Safemode Exception |
|GetPipeline       | Should Succeed.   |
