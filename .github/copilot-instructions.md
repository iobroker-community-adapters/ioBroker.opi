# ioBroker Adapter Development with GitHub Copilot

**Version:** 0.4.0
**Template Source:** https://github.com/DrozmotiX/ioBroker-Copilot-Instructions

This file contains instructions and best practices for GitHub Copilot when working on ioBroker adapter development.

## Project Context

You are working on an ioBroker adapter. ioBroker is an integration platform for the Internet of Things, focused on building smart home and industrial IoT solutions. Adapters are plugins that connect ioBroker to external systems, devices, or services.

### Adapter-Specific Context
- **Adapter Name**: opi
- **Primary Function**: OrangePI system monitoring and hardware resource tracking
- **Target Platform**: Linux-based OrangePI single-board computers
- **Monitoring Categories**: CPU, Memory, Network (eth0), eMMC storage, Swap, Temperature, Uptime, WLAN
- **Key Dependencies**: sync-exec for system command execution
- **Configuration**: Configurable monitoring modules with customizable intervals (default: 60 seconds)

### Hardware Integration Details
This adapter monitors OrangePI hardware resources by executing Linux system commands and parsing their output:
- **CPU monitoring**: Reading from `/sys/devices/system/cpu/cpu0/cpufreq/` and `/proc/loadavg`
- **Memory tracking**: Parsing `/proc/meminfo` and using `free` command
- **Network statistics**: Reading from `/sys/class/net/eth0/statistics/` and `/sys/class/net/wlan0/statistics/`
- **Storage monitoring**: Using `df` command for filesystem usage
- **Temperature readings**: Reading from `/sys/devices/virtual/thermal/thermal_zone0/temp`
- **System uptime**: Reading from `/proc/uptime`

### System Command Patterns
The adapter uses regular expressions to parse command output and post-processing functions to format values. Key patterns:
- File reads: `cat /sys/...` or `cat /proc/...`
- Command execution: Using sync-exec for synchronous system calls
- Value processing: Mathematical operations like division for unit conversion (e.g., `/1000` for temperature)
- Pattern matching: Complex regex patterns for multi-line command outputs

## Testing

### Unit Testing
- Use Jest as the primary testing framework for ioBroker adapters
- Create tests for all adapter main functions and helper methods
- Test error handling scenarios and edge cases
- Mock external API calls and hardware dependencies
- For adapters connecting to APIs/devices not reachable by internet, provide example data files to allow testing of functionality without live connections
- Example test structure:
  ```javascript
  describe('AdapterName', () => {
    let adapter;
    
    beforeEach(() => {
      // Setup test adapter instance
    });
    
    test('should initialize correctly', () => {
      // Test adapter initialization
    });
  });
  ```

### Integration Testing

**IMPORTANT**: Use the official `@iobroker/testing` framework for all integration tests. This is the ONLY correct way to test ioBroker adapters.

**Official Documentation**: https://github.com/ioBroker/testing

#### Framework Structure
Integration tests MUST follow this exact pattern:

```javascript
const path = require('path');
const { tests } = require('@iobroker/testing');

// Define test coordinates or configuration
const TEST_COORDINATES = '52.520008,13.404954'; // Berlin
const wait = ms => new Promise(resolve => setTimeout(resolve, ms));

// Use tests.integration() with defineAdditionalTests
tests.integration(path.join(__dirname, '..'), {
    defineAdditionalTests({ suite }) {
        suite('Test adapter with specific configuration', (getHarness) => {
            let harness;

            before(() => {
                harness = getHarness();
            });

            it('should configure and start adapter', function () {
                return new Promise(async (resolve, reject) => {
                    try {
                        harness = getHarness();
                        
                        // Get adapter object using promisified pattern
                        const obj = await new Promise((res, rej) => {
                            harness.objects.getObject('system.adapter.your-adapter.0', (err, o) => {
                                if (err) return rej(err);
                                res(o);
                            });
                        });
                        
                        if (!obj) {
                            return reject(new Error('Adapter object not found'));
                        }

                        // Configure adapter properties
                        Object.assign(obj.native, {
                            position: TEST_COORDINATES,
                            createCurrently: true,
                            createHourly: true,
                            createDaily: true,
                            // Add other configuration as needed
                        });

                        // Set the updated configuration
                        harness.objects.setObject(obj._id, obj);

                        console.log('✅ Step 1: Configuration written, starting adapter...');
                        
                        // Start adapter and wait
                        await harness.startAdapterAndWait();
                        
                        console.log('✅ Step 2: Adapter started');

                        // Wait for adapter to process data
                        const waitMs = 15000;
                        await wait(waitMs);

                        console.log('🔍 Step 3: Checking states after adapter run...');
                        
                        // Check if states exist
                        const states = await harness.objects.getObjectListAsync({ 
                            startkey: harness.adapterName + '.0.', 
                            endkey: harness.adapterName + '.0.\u9999' 
                        });
                        
                        if (states.rows && states.rows.length > 0) {
                            console.log(`Found ${states.rows.length} adapter objects`);
                            
                            // Check info.connection state if present
                            const connectionState = await harness.states.getStateAsync(
                                harness.adapterName + '.0.info.connection'
                            );
                            
                            if (connectionState) {
                                console.log('Connection state:', connectionState.val);
                            }
                            
                            resolve();
                        } else {
                            console.log('❌ No states found after adapter run');
                            reject(new Error('No adapter states created'));
                        }
                    } catch (error) {
                        console.error('Test failed with error:', error.message);
                        reject(error);
                    }
                });
            }).timeout(30000);
        });
    }
});
```

#### Key Points for ioBroker Testing Framework
1. **Always use the official framework**: `const { tests } = require('@iobroker/testing');`
2. **File location**: Place integration tests in `test/integration.js` or similar
3. **Test structure**: Use `tests.integration(path.join(__dirname, '..'))` as the entry point
4. **Extended tests**: Add specific tests using `defineAdditionalTests` callback
5. **Harness object**: Access adapter and system through the testing harness
6. **Async patterns**: Use proper promise handling with the testing framework
7. **State checking**: Always verify that your adapter creates expected states and objects

## Development Patterns

### ioBroker Adapter Structure
- **main.js**: Entry point with adapter initialization
- **io-package.json**: Adapter configuration and metadata
- **admin/**: Administrative interface files (HTML, CSS, JS)
- **lib/**: Utility libraries and helpers

### Initialization Pattern
```javascript
const utils = require(__dirname + '/lib/utils');

const adapter = utils.Adapter({
    name: 'adapter-name',
    
    ready: function () {
        // Adapter startup logic
        main();
    },
    
    stateChange: function (id, state) {
        // Handle state changes
    },
    
    unload: function (callback) {
        // Cleanup logic
        callback();
    }
});
```

### State Management
```javascript
// Create state objects
adapter.extendObject('state.name', {
    type: 'state',
    common: {
        name: 'State Name',
        type: 'number',
        role: 'value',
        read: true,
        write: false,
        unit: 'unit'
    },
    native: {}
});

// Set state values
adapter.setState('state.name', {
    val: value,
    ack: true
});
```

### System Command Execution (OPI-specific)
```javascript
const exec = require('child_process').execSync;

// Execute system command and parse output
function getSystemValue(command, regexp, postProcess) {
    try {
        const output = exec(command, { encoding: 'utf8' });
        const match = output.match(regexp);
        
        if (match) {
            let value = match[1];
            if (postProcess) {
                // Apply post-processing (e.g., unit conversion)
                value = eval(postProcess.replace('$1', value));
            }
            return value;
        }
    } catch (error) {
        adapter.log.error(`Command failed: ${command} - ${error.message}`);
    }
    return null;
}

// Example usage for temperature monitoring
const temperature = getSystemValue(
    'cat /sys/devices/virtual/thermal/thermal_zone0/temp',
    '(.*)',
    '$1/1000'
);
```

### Configuration Handling
```javascript
// Access adapter configuration
const config = adapter.config;

// Check if monitoring module is enabled
if (config.c_cpu) {
    // CPU monitoring is enabled
    monitorCPU();
}

// Use interval from configuration
const interval = config.interval || 60000; // Default 60 seconds
```

### Error Handling
- Use appropriate logging levels (error, warn, info, debug)
- Implement proper error handling for system commands
- Handle missing files or permission errors gracefully
- Log meaningful error messages with context

```javascript
try {
    const result = exec(command);
    // Process result
} catch (error) {
    if (error.code === 'ENOENT') {
        adapter.log.warn(`File not found: ${command}`);
    } else {
        adapter.log.error(`Command execution failed: ${error.message}`);
    }
}
```

### Cleanup and Resource Management
```javascript
function unload(callback) {
    try {
        // Clear intervals
        if (this.pollingTimer) {
            clearInterval(this.pollingTimer);
            this.pollingTimer = null;
        }
        
        // Clear timeouts
        if (this.connectionTimer) {
            clearTimeout(this.connectionTimer);
            this.connectionTimer = undefined;
        }
        // Close connections, clean up resources
        callback();
    } catch (e) {
        callback();
    }
}
```

## Code Style and Standards

- Follow JavaScript/TypeScript best practices
- Use async/await for asynchronous operations
- Implement proper resource cleanup in `unload()` method
- Use semantic versioning for adapter releases
- Include proper JSDoc comments for public methods

## CI/CD and Testing Integration

### GitHub Actions for API Testing
For adapters with external API dependencies, implement separate CI/CD jobs:

```yaml
# Tests API connectivity with demo credentials (runs separately)
demo-api-tests:
  if: contains(github.event.head_commit.message, '[skip ci]') == false
  
  runs-on: ubuntu-22.04
  
  steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Use Node.js 20.x
      uses: actions/setup-node@v4
      with:
        node-version: 20.x
        cache: 'npm'
        
    - name: Install dependencies
      run: npm ci
      
    - name: Run demo API tests
      run: npm run test:integration-demo
```

### CI/CD Best Practices
- Run credential tests separately from main test suite
- Use ubuntu-22.04 for consistency
- Don't make credential tests required for deployment
- Provide clear failure messages for API connectivity issues
- Use appropriate timeouts for external API calls (120+ seconds)

### Package.json Script Integration
Add dedicated script for credential testing:
```json
{
  "scripts": {
    "test:integration-demo": "mocha test/integration-demo --exit"
  }
}
```

### Practical Example: Complete API Testing Implementation
Here's a complete example based on lessons learned from the Discovergy adapter:

#### test/integration-demo.js
```javascript
const path = require("path");
const { tests } = require("@iobroker/testing");

// Helper function to encrypt password using ioBroker's encryption method
async function encryptPassword(harness, password) {
    const systemConfig = await harness.objects.getObjectAsync("system.config");
    
    if (!systemConfig || !systemConfig.native || !systemConfig.native.secret) {
        throw new Error("Could not retrieve system secret for password encryption");
    }
    
    const secret = systemConfig.native.secret;
    let result = '';
    for (let i = 0; i < password.length; ++i) {
        result += String.fromCharCode(secret[i % secret.length].charCodeAt(0) ^ password.charCodeAt(i));
    }
    
    return result;
}

// Run integration tests with demo credentials
tests.integration(path.join(__dirname, ".."), {
    defineAdditionalTests({ suite }) {
        suite("API Testing with Demo Credentials", (getHarness) => {
            let harness;
            
            before(() => {
                harness = getHarness();
            });

            it("Should connect to API and initialize with demo credentials", async () => {
                console.log("Setting up demo credentials...");
                
                if (harness.isAdapterRunning()) {
                    await harness.stopAdapter();
                }
                
                const encryptedPassword = await encryptPassword(harness, "demo_password");
                
                await harness.changeAdapterConfig("your-adapter", {
                    native: {
                        username: "demo@provider.com",
                        password: encryptedPassword,
                        // other config options
                    }
                });

                console.log("Starting adapter with demo credentials...");
                await harness.startAdapter();
                
                // Wait for API calls and initialization
                await new Promise(resolve => setTimeout(resolve, 60000));
                
                const connectionState = await harness.states.getStateAsync("your-adapter.0.info.connection");
                
                if (connectionState && connectionState.val === true) {
                    console.log("✅ SUCCESS: API connection established");
                    return true;
                } else {
                    throw new Error("API Test Failed: Expected API connection to be established with demo credentials. " +
                        "Check logs above for specific API errors (DNS resolution, 401 Unauthorized, network issues, etc.)");
                }
            }).timeout(120000);
        });
    }
});
```

### OPI Adapter Testing Considerations
- Mock system file reads for unit tests (no actual hardware needed)
- Test command parsing logic with sample outputs
- Validate regex patterns against known command outputs
- Test error scenarios (missing files, permission errors)
- Verify post-processing calculations for unit conversions