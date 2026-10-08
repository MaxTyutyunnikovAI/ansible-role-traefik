# Local Ansible workspace (checked into git on purpose)

`.ansible/roles/traefik` is a relative symlink to the repository root so that
`molecule/default/converge.yml` (`- role: traefik`) and `ansible-lint`'s
syntax-check resolve the role when this repo is checked out stand-alone,
without installing it from Galaxy first. The pattern follows the existing
tracked `.ansible/roles/w0.traefik` symlink. Everything else under `.ansible/`
(collections, modules, caches) remains ignored via `.gitignore`.
