# AI & Community Notice

## About This Repository

This repository is a **community-driven effort** to automate the migration process from RPM-based installations of Red Hat Ansible Automation Platform (AAP) to containerized or OpenShift deployments. It aims to provide a practical, reusable set of Ansible playbooks that reduce the manual effort involved in following the [official AAP 2.6 Migration Guide](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/ansible_automation_platform_migration/index).

## AI-Assisted Development

Portions of the code in this repository have been generated with the assistance of AI tools. AI has been used throughout the development process, including:

- Scaffolding playbooks, roles, and task files
- Generating inventory templates and variable definitions
- Drafting documentation

All AI-generated content has been reviewed for correctness, but users should exercise their own judgment and thoroughly test playbooks in non-production environments before relying on them for actual migrations.

## Contributing

Both human-authored and AI-assisted pull requests are welcome. Whether you write code by hand, use AI tools to help draft changes, or combine both approaches, contributions that improve the quality, coverage, and reliability of these migration playbooks are appreciated.

When submitting a PR, please:

- Test your changes against a representative environment when possible
- Clearly describe what the change does and why
- Follow existing conventions (FQCN for modules, decomposed task files, confirmation prompts for destructive operations)

## Disclaimer

> **This repository is not supported by Red Hat.** It is an independent community project. Any use of the contents of this repository is at your own risk and **will not be supported by Red Hat** through any Red Hat support channel, subscription, or agreement.
>
> Red Hat Ansible Automation Platform is a product of Red Hat, Inc. This project is not affiliated with, endorsed by, or sponsored by Red Hat.

For official migration guidance, refer to the [AAP 2.6 Migration Guide](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/ansible_automation_platform_migration/index) published by Red Hat.

## License

This project is licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for details.
