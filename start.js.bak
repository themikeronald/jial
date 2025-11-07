#!/usr/bin/env node
// start.js - Smart launcher for NOX Engine Bot
// Downloads latest bot from server and runs with optimal heap size

require('dotenv').config({ quiet: true });
const { spawn } = require('child_process');
const os = require('os');
const v8 = require('v8');
const chalk = require('chalk');
const axios = require('axios');
const fs = require('fs');
const path = require('path');
const crypto = require('crypto');

// Configuration
const LICENSE_SERVER_URL = process.env.LICENSE_SERVER_URL || 'http://localhost:3000';
// Use .nox-cache directory in project instead of system temp for better module resolution
const TEMP_DIR = path.join(__dirname, '.nox-cache');
const CACHE_FILE = path.join(TEMP_DIR, '.cache.json');

// Calculate optimal heap size
function calculateOptimalHeap() {
    const totalRAM = Math.round(os.totalmem() / (1024 ** 3)); // GB
    const heapStats = v8.getHeapStatistics();
    const currentHeapLimit = Math.round(heapStats.heap_size_limit / (1024 * 1024)); // MB
    
    // Optimal: 85% of system RAM
    const optimalHeapSize = Math.floor((totalRAM * 1024) * 0.85); // MB
    
    return {
        totalRAM,
        currentHeapLimit,
        optimalHeapSize,
        needsOptimization: (optimalHeapSize - currentHeapLimit) > 1000
    };
}

// Get file hash
function getFileHash(filePath) {
    if (!fs.existsSync(filePath)) return null;
    const fileBuffer = fs.readFileSync(filePath);
    return crypto.createHash('sha256').update(fileBuffer).digest('hex');
}

// Read cache
function readCache() {
    try {
        if (fs.existsSync(CACHE_FILE)) {
            return JSON.parse(fs.readFileSync(CACHE_FILE, 'utf8'));
        }
    } catch (error) {
        // Ignore cache errors
    }
    return null;
}

// Write cache
function writeCache(data) {
    try {
        fs.mkdirSync(TEMP_DIR, { recursive: true });
        fs.writeFileSync(CACHE_FILE, JSON.stringify(data, null, 2));
    } catch (error) {
        console.error(chalk.yellow('⚠️  Failed to write cache:', error.message));
    }
}

// Check if update is needed
async function checkUpdate() {
    try {
        const cache = readCache();
        const botPath = path.join(TEMP_DIR, 'bot.js');
        const currentHash = getFileHash(botPath);
        
        const response = await axios.get(`${LICENSE_SERVER_URL}/api/bot/check-update`, {
            params: {
                version: cache?.version,
                hash: currentHash
            },
            timeout: 5000
        });
        
        return response.data;
    } catch (error) {
        if (error.code === 'ECONNREFUSED') {
            console.log(chalk.yellow('⚠️  License server offline, using cached bot'));
        } else {
            console.error(chalk.yellow('⚠️  Update check failed:', error.message));
        }
        return { updateAvailable: false };
    }
}

// Download bot from server
async function downloadBot() {
    try {
        console.log(chalk.cyan('📦 Downloading latest bot from server...'));
        
        const response = await axios.get(`${LICENSE_SERVER_URL}/api/bot/download`, {
            responseType: 'arraybuffer',
            timeout: 30000
        });
        
        // Get metadata from headers
        const version = response.headers['x-bot-version'] || 'unknown';
        const hash = response.headers['x-file-hash'];
        
        // Save to temp directory
        fs.mkdirSync(TEMP_DIR, { recursive: true });
        const botPath = path.join(TEMP_DIR, 'bot.js');
        fs.writeFileSync(botPath, response.data);
        
        // Verify hash
        const downloadedHash = getFileHash(botPath);
        if (hash && downloadedHash !== hash) {
            throw new Error('File integrity check failed');
        }
        
        // Save cache
        writeCache({
            version,
            hash: downloadedHash,
            downloadedAt: new Date().toISOString(),
            fileSize: response.data.length
        });
        
        console.log(chalk.green(`✅ Downloaded v${version} (${(response.data.length / 1024).toFixed(2)} KB)`));
        return botPath;
        
    } catch (error) {
        console.error(chalk.red('❌ Download failed:', error.message));
        throw error;
    }
}

// Cleanup temp directory
function cleanup() {
    try {
        const botPath = path.join(TEMP_DIR, 'bot.js');
        if (fs.existsSync(botPath)) {
            fs.unlinkSync(botPath);
            console.log(chalk.gray('\n🧹 Cleaned up temporary files'));
        }
    } catch (error) {
        console.error(chalk.yellow('⚠️  Cleanup failed:', error.message));
    }
}

// Main launcher
async function launchBot() {
    const heap = calculateOptimalHeap();
    
    console.log('');
    console.log(chalk.cyan('🚀 NOX Engine Launcher'));
    console.log(chalk.gray('   ────────────────────────────────────────'));
    console.log(chalk.gray('   System RAM    : ') + chalk.white(`${heap.totalRAM} GB`));
    console.log(chalk.gray('   Current Heap  : ') + chalk.white(`${heap.currentHeapLimit} MB`) + chalk.gray(` (~${(heap.currentHeapLimit/1024).toFixed(1)} GB)`));
    console.log(chalk.gray('   Optimal Heap  : ') + chalk.green(`${heap.optimalHeapSize} MB`) + chalk.gray(` (~${(heap.optimalHeapSize/1024).toFixed(1)} GB)`));
    console.log(chalk.gray('   ────────────────────────────────────────'));
    console.log('');
    
    try {
        // Check for updates
        console.log(chalk.cyan('🔍 Checking for updates...'));
        const updateInfo = await checkUpdate();
        
        let botPath;
        
        if (updateInfo.updateAvailable) {
            console.log(chalk.yellow(`⚡ Update available: v${updateInfo.currentVersion} → v${updateInfo.latestVersion}`));
            botPath = await downloadBot();
        } else {
            // Use cached bot
            botPath = path.join(TEMP_DIR, 'bot.js');
            
            if (!fs.existsSync(botPath)) {
                console.log(chalk.yellow('⚠️  No cached bot found, downloading...'));
                botPath = await downloadBot();
            } else {
                const cache = readCache();
                console.log(chalk.green(`✅ Using cached bot v${cache?.version || 'unknown'}`));
            }
        }
        
        console.log('');
        console.log(chalk.cyan('🚀 Starting bot with optimal heap...'));
        console.log('');
        
        // Read bot code from downloaded file
        const botCode = fs.readFileSync(botPath, 'utf8');
        
        // Execute bot code inline in this process
        // This way bot has access to all node_modules from start.js context
        try {
            // Remove shebang if exists
            const cleanCode = botCode.replace(/^#!.*\n/, '');
            
            // Execute the bot code
            eval(cleanCode);
            
            // Note: After eval, bot.js code will run immediately
            // Setup cleanup handlers
            process.on('exit', () => {
                cleanup();
            });
            
            process.on('SIGINT', () => {
                console.log(chalk.yellow('\n\n⚠️  Interrupted by user'));
                cleanup();
                process.exit(0);
            });
            
            process.on('SIGTERM', () => {
                cleanup();
                process.exit(0);
            });
            
        } catch (error) {
            console.error(chalk.red('❌ Bot execution error:'), error.message);
            console.error(error.stack);
            cleanup();
            process.exit(1);
        }
        
    } catch (error) {
        console.error(chalk.red('\n❌ Launcher error:'), error.message);
        console.log(chalk.yellow('\n💡 Tip: Check your LICENSE_SERVER_URL in .env'));
        console.log(chalk.gray(`   Current: ${LICENSE_SERVER_URL}`));
        process.exit(1);
    }
}

// Run launcher
launchBot();
