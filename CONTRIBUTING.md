# Contributing to AetherWeb3 Multi-EVM Gateway

We welcome contributions to the AetherWeb3 Multi-EVM Gateway! This document provides guidelines for contributing to the project.

## Code of Conduct

By participating in this project, you agree to abide by our Code of Conduct:

### Our Pledge
- Be respectful and inclusive
- Welcome newcomers and help them learn
- Focus on what's best for the community
- Show empathy towards other community members

### Unacceptable Behavior
- Harassment or discrimination of any kind
- Trolling, insulting, or derogatory comments
- Public or private harassment
- Publishing others' private information without permission

## How to Contribute

### Reporting Issues

Before creating an issue, please:
1. **Search existing issues** to avoid duplicates
2. **Use issue templates** when available
3. **Provide detailed information** including:
   - Operating system and version
   - Node.js version
   - Steps to reproduce
   - Expected vs actual behavior
   - Error messages and stack traces

### Feature Requests

We welcome feature requests! Please:
1. **Check if the feature already exists** or is planned
2. **Describe the use case** and problem it solves
3. **Propose a solution** if you have ideas
4. **Consider backward compatibility** and breaking changes

### Pull Requests

#### Before You Start
1. **Discuss major changes** in an issue first
2. **Fork the repository** and create a feature branch
3. **Set up your development environment** following our [Development Guide](docs/development/guide.md)

#### Development Process

1. **Create a Feature Branch**
```bash
git checkout -b feature/your-feature-name
# or
git checkout -b bugfix/issue-number-description
```

2. **Follow Coding Standards**
- Use ESLint and Prettier configurations
- Write meaningful commit messages
- Add tests for new functionality
- Update documentation as needed

3. **Commit Message Format**
Follow [Conventional Commits](https://conventionalcommits.org/):
```
type(scope): description

[optional body]

[optional footer]
```

Types:
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes (formatting, etc.)
- `refactor`: Code refactoring
- `test`: Adding or updating tests
- `chore`: Maintenance tasks

Examples:
```
feat(api): add Polygon network support
fix(auth): resolve token validation issue
docs(readme): update installation instructions
```

4. **Testing Requirements**
- All tests must pass: `npm test`
- Maintain code coverage above 80%
- Add integration tests for new API endpoints
- Include end-to-end tests for major features

5. **Documentation Requirements**
- Update API documentation for new endpoints
- Add examples for new features
- Update README if needed
- Include JSDoc comments for new functions

#### Pull Request Process

1. **Create Pull Request**
- Use the pull request template
- Link to related issues
- Provide clear description of changes
- Include screenshots for UI changes

2. **Code Review Process**
- At least one maintainer review required
- Address feedback promptly
- Keep discussions focused and professional
- Update code based on feedback

3. **Merge Requirements**
- All CI checks must pass
- Code review approval required
- No merge conflicts
- Documentation updated

### Security Contributions

For security-related issues:
1. **DO NOT** create public issues for security vulnerabilities
2. **Email security@nibertinvestments.com** with details
3. **Include proof of concept** if available
4. **Allow reasonable time** for response before disclosure

## Development Guidelines

### Code Quality Standards

1. **JavaScript/Node.js**
- Use ES6+ features consistently
- Follow ESLint configuration
- Prefer async/await over callbacks
- Use meaningful variable and function names

2. **Error Handling**
- Always handle errors appropriately
- Use custom error classes when needed
- Log errors with sufficient context
- Don't expose sensitive information in error messages

3. **Performance**
- Optimize for scalability
- Use caching where appropriate
- Monitor memory usage
- Implement proper connection pooling

4. **Security**
- Validate all inputs
- Use parameterized queries
- Implement rate limiting
- Follow OWASP best practices

### Testing Guidelines

1. **Unit Tests**
- Test individual functions and modules
- Mock external dependencies
- Cover edge cases and error conditions
- Aim for 100% code coverage on critical paths

2. **Integration Tests**
- Test API endpoints end-to-end
- Use test databases when needed
- Verify error responses
- Test rate limiting behavior

3. **Load Tests**
- Test performance under load
- Verify scalability characteristics
- Identify bottlenecks
- Test failover scenarios

### Documentation Standards

1. **Code Documentation**
- Use JSDoc for all public functions
- Include parameter types and descriptions
- Document return values and exceptions
- Provide usage examples

2. **API Documentation**
- Update OpenAPI/Swagger specs
- Include request/response examples
- Document error codes and meanings
- Provide integration examples

3. **User Documentation**
- Write clear, step-by-step guides
- Include code examples in multiple languages
- Test documentation with real users
- Keep documentation up-to-date

## Contribution Types

### Code Contributions
- New features and enhancements
- Bug fixes and patches
- Performance optimizations
- Security improvements

### Documentation Contributions
- API documentation improvements
- User guide enhancements
- Code example additions
- Translation efforts

### Community Contributions
- Issue triage and support
- Code reviews
- Testing and QA
- Community management

## Recognition

Contributors will be recognized:
- In the CONTRIBUTORS.md file
- In release notes for significant contributions
- On our website and social media
- Through GitHub contributor statistics

## Resources

### Development Resources
- [Development Guide](docs/development/guide.md)
- [API Reference](docs/api/reference.md)
- [Architecture Overview](docs/architecture/overview.md)
- [Security Best Practices](docs/security/best-practices.md)

### Community Resources
- [Discord Community](#) (coming soon)
- [GitHub Discussions](https://github.com/nibertinvestments/multi-evm-gateway-8511/discussions)
- [Stack Overflow Tag](#) (coming soon)

### Contact Information
- **General Questions**: support@nibertinvestments.com
- **Security Issues**: security@nibertinvestments.com
- **Business Inquiries**: sales@nibertinvestments.com

## License

By contributing to this project, you agree that your contributions will be licensed under the [MIT License](LICENSE).

## Getting Help

If you need help with contributing:
1. **Read the documentation** thoroughly
2. **Search existing issues** and discussions
3. **Ask in discussions** for general questions
4. **Email support** for specific technical issues

Thank you for contributing to AetherWeb3! 🚀

---

*Last Updated: January 2025*