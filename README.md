// Configuration
const API_KEY = 'your-anthropic-api-key-here'; // Replace with your actual API key
const API_URL = 'https://api.anthropic.com/v1/messages';

// DOM Elements
const userInput = document.getElementById('userInput');
const submitBtn = document.getElementById('submitBtn');
const errorMessage = document.getElementById('errorMessage');
const loadingSpinner = document.getElementById('loadingSpinner');
const resultContainer = document.getElementById('resultContainer');
const historyList = document.getElementById('historyList');
const clearHistoryBtn = document.getElementById('clearHistoryBtn');

// State
let calculationHistory = [];

// Load history from localStorage
function loadHistory() {
    const saved = localStorage.getItem('calculationHistory');
    if (saved) {
        calculationHistory = JSON.parse(saved);
        renderHistory();
    }
}

// Save history to localStorage
function saveHistory() {
    localStorage.setItem('calculationHistory', JSON.stringify(calculationHistory));
}

// Event listeners
submitBtn.addEventListener('click', handleCalculate);
userInput.addEventListener('keypress', (e) => {
    if (e.key === 'Enter') {
        handleCalculate();
    }
});
clearHistoryBtn.addEventListener('click', clearHistory);

// Main calculation handler
async function handleCalculate() {
    const question = userInput.value.trim();

    if (!question) {
        showError('Please enter a question');
        return;
    }

    if (!API_KEY || API_KEY === 'your-anthropic-api-key-here') {
        showError('API key not configured. Please add your Anthropic API key to script.js');
        return;
    }

    clearError();
    showLoading(true);
    submitBtn.disabled = true;

    try {
        const result = await callAI(question);
        displayResult(result, question);
        addToHistory(result, question);
    } catch (error) {
        showError(`Error: ${error.message}`);
    } finally {
        showLoading(false);
        submitBtn.disabled = false;
        userInput.value = '';
    }
}

// Call Anthropic API
async function callAI(question) {
    const systemPrompt = `You are a helpful AI calculator assistant. When a user asks a math question:
1. Identify the mathematical expression they're asking about
2. Show the mathematical expression clearly
3. Calculate the correct answer
4. Provide a brief explanation if needed

Respond ONLY in valid JSON format with these exact fields:
{
  "expression": "the mathematical expression (e.g., '25 + 17')",
  "answer": "the numerical answer",
  "explanation": "brief explanation of the calculation"
}

Do not include markdown, code blocks, or any other formatting. Just valid JSON.`;

    const response = await fetch(API_URL, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'x-api-key': API_KEY,
            'anthropic-version': '2023-06-01'
        },
        body: JSON.stringify({
            model: 'claude-3-5-sonnet-20241022',
            max_tokens: 1024,
            system: systemPrompt,
            messages: [
                {
                    role: 'user',
                    content: question
                }
            ]
        })
    });

    if (!response.ok) {
        const errorData = await response.json();
        throw new Error(errorData.error?.message || `API error: ${response.status}`);
    }

    const data = await response.json();
    const content = data.content[0].text;

    // Parse the JSON response
    const result = JSON.parse(content);
    return result;
}

// Display results
function displayResult(result, question) {
    document.getElementById('userQuestion').textContent = question;
    document.getElementById('mathExpression').textContent = result.expression;
    document.getElementById('answer').textContent = result.answer;

    if (result.explanation) {
        document.getElementById('explanation').textContent = result.explanation;
        document.getElementById('explanationContainer').classList.remove('hidden');
    } else {
        document.getElementById('explanationContainer').classList.add('hidden');
    }

    resultContainer.classList.remove('hidden');
}

// Add to history
function addToHistory(result, question) {
    const historyItem = {
        id: Date.now(),
        question: question,
        expression: result.expression,
        answer: result.answer,
        timestamp: new Date().toLocaleString()
    };

    calculationHistory.unshift(historyItem);
    if (calculationHistory.length > 20) {
        calculationHistory.pop();
    }

    saveHistory();
    renderHistory();
}

// Render history list
function renderHistory() {
    if (calculationHistory.length === 0) {
        historyList.innerHTML = '<p class="empty-message">No calculations yet</p>';
        return;
    }

    historyList.innerHTML = calculationHistory.map(item => `
        <div class="history-item" onclick="loadFromHistory(${item.id})">
            <div class="history-question">${escapeHtml(item.question)}</div>
            <div class="history-answer">${escapeHtml(item.answer)}</div>
            <small style="color: #999;">${item.timestamp}</small>
        </div>
    `).join('');
}

// Load calculation from history
function loadFromHistory(id) {
    const item = calculationHistory.find(h => h.id === id);
    if (item) {
        displayResult(
            {
                expression: item.expression,
                answer: item.answer,
                explanation: ''
            },
            item.question
        );
    }
}

// Clear history
function clearHistory() {
    if (confirm('Are you sure you want to clear all history?')) {
        calculationHistory = [];
        saveHistory();
        renderHistory();
    }
}

// Utility functions
function showError(message) {
    errorMessage.textContent = message;
    errorMessage.classList.add('show');
}

function clearError() {
    errorMessage.classList.remove('show');
    errorMessage.textContent = '';
}

function showLoading(show) {
    if (show) {
        loadingSpinner.classList.remove('hidden');
        resultContainer.classList.add('hidden');
    } else {
        loadingSpinner.classList.add('hidden');
    }
}

function copyToClipboard(elementId) {
    const element = document.getElementById(elementId);
    const text = element.textContent;

    navigator.clipboard.writeText(text).then(() => {
        const btn = event.target;
        const originalText = btn.textContent;
        btn.textContent = '✓ Copied!';
        setTimeout(() => {
            btn.textContent = originalText;
        }, 2000);
    });
}

function escapeHtml(text) {
    const div = document.createElement('div');
    div.textContent = text;
    return div.innerHTML;
}

// Initialize on page load
window.addEventListener('DOMContentLoaded', loadHistory);
# ai-calculator
An AI-powered calculator that processes natural language math expressions
