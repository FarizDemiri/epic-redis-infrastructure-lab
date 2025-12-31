# Contributing to Epic Redis Infrastructure Lab

Thank you for your interest in this project! While this is primarily a portfolio demonstration project, contributions that improve the documentation, fix issues, or enhance the examples are welcome.

## How to Contribute

### Reporting Issues

If you find a bug, security issue, or documentation error:

1. Check if the issue already exists in the Issues tab
2. Create a new issue with:
   - Clear, descriptive title
   - Steps to reproduce (for bugs)
   - Expected vs actual behavior
   - Your environment (OS, Redis version, AWS region)

### Suggesting Enhancements

For new features or improvements:

1. Open an issue describing:
   - The problem you're trying to solve
   - Your proposed solution
   - Why this would benefit others using this project

### Pull Requests

1. **Fork the repository**
2. **Create a feature branch:** `git checkout -b feature/your-feature-name`
3. **Make your changes:**
   - Follow existing code style
   - Update documentation if needed
   - Test your changes thoroughly
4. **Commit with clear messages:** `git commit -m "Add feature: X"`
5. **Push to your fork:** `git push origin feature/your-feature-name`
6. **Open a Pull Request** with:
   - Description of changes
   - Why the changes are needed
   - Any related issue numbers

## Code Style

- **Markdown:** Follow existing formatting (headers, code blocks, lists)
- **Shell Scripts:** Use consistent indentation (2 spaces)
- **Configuration Files:** Maintain YAML/JSON structure
- **Comments:** Explain *why*, not just *what*

## Documentation Standards

- **Runbooks:** Include exact commands with expected outputs
- **Architecture Docs:** Use Mermaid diagrams where possible
- **Code Examples:** Add comments explaining healthcare context

## Testing

Before submitting:

- ✅ Verify all commands work on a fresh AWS account
- ✅ Check that documentation is accurate
- ✅ Ensure no sensitive data (IPs, keys, passwords) is committed
- ✅ Test on Ubuntu 24.04 LTS (the project's base OS)

## Areas for Contribution

**Good first contributions:**

- Fixing typos or broken links
- Improving error messages
- Adding more Epic use case examples
- Expanding troubleshooting sections

**Advanced contributions:**

- Terraform/CloudFormation templates
- Ansible playbooks for automation
- Additional monitoring dashboards
- Performance benchmarking scripts

## Security

**Do NOT include:**

- Real AWS credentials or keys
- Actual hospital/patient data
- Production IP addresses
- Secrets or passwords

If you discover a security vulnerability, please email privately rather than opening a public issue.

## Questions?

Open an issue with the `question` label, and I'll respond as soon as possible.

---

**Note:** This project is maintained as a portfolio piece. Response times may vary, but all contributions are appreciated!
