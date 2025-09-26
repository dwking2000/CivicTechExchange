# Node.js v16.20 Upgrade Implementation Plan

**GitHub Issue**: [#1087 - Node.js v16.20 is no longer supported](https://github.com/DemocracyLab/CivicTechExchange/issues/1087)

## Executive Summary

**Issue**: Node.js v16.20.x reached End of Life in September 2023 and needs to be upgraded to a supported LTS version.

**Current State**: The project uses Node.js 16.20.x with outdated frontend tooling (Webpack 4, React 16.11, Babel 7.12) that has **compatibility issues** with modern Node.js versions.

**Recommended Approach**: **Incremental upgrade strategy** due to significant technical debt in frontend toolchain.

---

## Current Environment Analysis

### Node.js Version Status
- **Current**: Node.js 16.20.x (EOL September 2023) ⚠️
- **Local Development**: Node.js 24.6.0 (already using newer version)
- **Production/Docker**: Node.js 16 (via `nikolaik/python-nodejs:python3.10-nodejs16`)
- **Docker Internal**: Node.js 12.16.0 (even older!) 🔴

### Critical Compatibility Issues

1. **Webpack 4.42.0 + Node.js 20/22**: ❌ **INCOMPATIBLE**
   - OpenSSL 3.0 changes break MD4 algorithm usage
   - Webpack 4 has hard-coded MD4 usage that cannot be configured
   - Causes `ERR_OSSL_EVP_UNSUPPORTED` errors

2. **Frontend Dependencies**: Multiple compatibility blockers
   - React 16.11.0: Works with Node.js 20/22 but toolchain issues
   - Flow 0.75.0: Abandoned by Facebook, poor Node.js 20+ support
   - Babel 7.12.x: Older version may have compatibility issues

3. **Build Process**: Manual webpack + Django integration
   - No Hot Module Replacement
   - Slow development feedback loop
   - Complex deployment pipeline

---

## Implementation Strategy

### 🎯 **Option A: Conservative Incremental Upgrade** (RECOMMENDED)

**Target**: Node.js 18.20.x LTS (Hydrogen)
**Timeline**: 4-6 weeks
**Risk**: Low

#### Phase 1: Infrastructure Update (Week 1-2)
```bash
# Update Docker base image
FROM nikolaik/python-nodejs:python3.10-nodejs18

# Update package.json
"engines": {
  "node": "18.20.x"
}
```

#### Phase 2: Dependency Compatibility Testing (Week 2-3)
- Test current build with Node.js 18.20.x using legacy OpenSSL provider
- Add temporary workaround: `NODE_OPTIONS='--openssl-legacy-provider'`
- Verify all npm scripts work correctly
- Test Django static file collection

#### Phase 3: Production Deployment (Week 3-4)
- Update Dockerfile and docker-compose.yml
- Deploy to staging environment with Node.js 18
- Monitor for compatibility issues
- Rollback plan: revert Docker image version

#### Phase 4: Cleanup and Monitoring (Week 5-6)
- Remove legacy OpenSSL provider flag once confirmed stable
- Update documentation and CI/CD
- Monitor performance and error logs

---

### ⚡ **Option B: Modern Stack Migration**

**Target**: Node.js 22.x LTS + Webpack 5 + Modern Tooling
**Timeline**: 8-12 weeks
**Risk**: High (but better long-term outcome)

#### Major Changes Required:
1. **Webpack 4 → 5**: Breaking changes, config updates needed
2. **Node.js polyfills**: Webpack 5 removed automatic polyfills
3. **Build system**: Opportunity to add HMR, improve DX
4. **Dependencies**: Update React, Babel, all dev dependencies

This option aligns with the **Frontend Modernization Assessment** recommendations but is more complex.

---

## Recommended Implementation Plan

### 🚀 **Phase 1: Node.js 18.20.x Upgrade** (IMMEDIATE)

**Week 1: Local Environment Setup**
```bash
# Test current setup with Node.js 18
nvm install 18.20.4
nvm use 18.20.4
npm install
NODE_OPTIONS='--openssl-legacy-provider' npm run dev
```

**Week 2: Docker and Configuration Updates**
```dockerfile
# Update Dockerfile
FROM nikolaik/python-nodejs:python3.10-nodejs18

# Update docker-compose.yml Node version
ENV NODE_VERSION 18.20.4
```

```json
// Update package.json
{
  "engines": {
    "node": "18.20.x"
  },
  "scripts": {
    "build": "NODE_OPTIONS='--openssl-legacy-provider' webpack --config webpack.prod.js && npm run buildtask:collectstatic",
    "dev": "NODE_OPTIONS='--openssl-legacy-provider' webpack --config webpack.dev.js && npm run buildtask:collectstatic"
  }
}
```

**Week 3-4: Testing and Deployment**
- Comprehensive testing on staging
- Monitor build performance and error rates
- Production deployment with rollback plan

### 🔮 **Phase 2: Future Modernization** (OPTIONAL)

After Node.js 18 is stable, consider frontend modernization:
- Webpack 4 → 5 migration
- React 16 → 18 upgrade
- Modern build tooling (Vite, etc.)
- TypeScript migration (replace Flow)

---

## Risk Assessment & Mitigation

### High Risk Items ⚠️
1. **Webpack 4 + Node.js 18**: Still requires legacy OpenSSL provider
2. **Docker Multi-stage Issues**: Base image compatibility
3. **Production Deployment**: Multiple moving parts

### Mitigation Strategies
1. **Gradual Rollout**: Test extensively on staging first
2. **Rollback Plan**: Keep Node.js 16 Docker images available
3. **Monitoring**: Enhanced error logging during transition
4. **Feature Flags**: Disable non-critical features if issues arise

### Low Risk Items ✅
1. **Django/Python Backend**: No direct Node.js dependencies
2. **React Runtime**: Works fine with Node.js 18-22
3. **Database**: No impact on PostgreSQL operations

---

## Testing Strategy

### Pre-Deployment Testing
```bash
# 1. Verify build process
NODE_OPTIONS='--openssl-legacy-provider' npm run build

# 2. Test development workflow
NODE_OPTIONS='--openssl-legacy-provider' npm run dev

# 3. Run frontend tests
NODE_OPTIONS='--openssl-legacy-provider' npm test

# 4. Test Django integration
python manage.py collectstatic --noinput
python manage.py runserver
```

### Production Validation
- [ ] Frontend assets build successfully
- [ ] Static file collection works
- [ ] No JavaScript runtime errors
- [ ] Performance metrics unchanged
- [ ] Error rates within acceptable limits

---

## Rollback Strategy

### Immediate Rollback (< 1 hour)
```dockerfile
# Revert Dockerfile
FROM nikolaik/python-nodejs:python3.10-nodejs16
```

### Configuration Rollback
```json
// Revert package.json
{
  "engines": {
    "node": "16.20.x"
  }
}
```

### Emergency Procedures
1. **Database**: No rollback needed (no DB changes)
2. **Static Files**: Re-run collectstatic with old version
3. **Docker**: Keep previous image tagged for instant rollback
4. **Monitoring**: Set up alerts for build failures

---

## Success Metrics

### Technical Metrics
- [ ] Build time: ≤ current performance
- [ ] Bundle size: No significant increase
- [ ] Error rate: < 0.1% increase
- [ ] Page load times: No degradation

### Security Metrics
- [ ] Node.js security vulnerabilities: 0 critical
- [ ] Dependency audit: All high/critical issues resolved
- [ ] OpenSSL version: Using supported version

---

## Next Steps

### Immediate Actions (This Week)
1. **Test locally** with Node.js 18.20.x + legacy OpenSSL provider
2. **Update Docker configuration** files
3. **Prepare staging environment** for testing

### Short Term (Next Month)
1. **Deploy to staging** and validate functionality
2. **Performance testing** under load
3. **Production deployment** with monitoring

### Long Term (Next Quarter)
1. **Plan frontend modernization** (Webpack 5 migration)
2. **Evaluate modern tooling** options (Vite, TypeScript)
3. **Consider Node.js 22** upgrade once toolchain is modernized

---

## Questions & Considerations

1. **Deployment Timing**: Is there a preferred maintenance window for this change?

2. **Monitoring Requirements**: What specific metrics should we track during the transition?

3. **Team Coordination**: Are there any ongoing frontend development efforts that would be impacted?

4. **Future Planning**: Should we prioritize this upgrade over the frontend modernization roadmap?

5. **Infrastructure**: Are there any Heroku or deployment platform considerations for Node.js version changes?

---

## Configuration Files to Update

### Files Requiring Changes
- `Dockerfile` - Update base image to nodejs18
- `docker-compose.yml` - Update NODE_VERSION environment variable
- `package.json` - Update engines.node specification and add NODE_OPTIONS to scripts
- `.nvmrc` - Create file with "18.20.4" (if using nvm)
- Documentation - Update setup instructions

### Files to Monitor
- `webpack.common.js` - May need adjustments for OpenSSL compatibility
- `webpack.prod.js` - Monitor for build optimization issues
- `webpack.dev.js` - Ensure development builds continue working
- `requirements.txt` - No changes needed (Python/Django unaffected)

---

## Related Issues & Dependencies

**Depends On**: None (standalone upgrade)
**Blocks**: Future frontend modernization efforts
**Related**: Frontend Modernization Assessment recommendations
**Priority**: High (security/maintenance)

---

**Recommendation**: Start with **Option A (Conservative Upgrade to Node.js 18.20.x)** to address the immediate security concern, then plan frontend modernization as a separate initiative for better long-term outcomes.

---

*Document Version: 1.0*
*Created: 2025-09-04*
*Status: Planning Phase*
*Estimated Effort: 4-6 weeks*