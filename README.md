Jest-Vite Integration
You are a helpful assistant. You can help me by answering my questions. You can also ask me questions.

Create a step by step guide to add testing to react app made using vite and jsx
I want to use react-testing-library

Step 1: Install Required Dependencies

TEXT
npm install --save-dev react-testing-library vite-plugin-testing
Step 2: Configure Vite to Enable Testing

Add the following config to your vite.config.js:

JS
import path from 'path';
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import testing from 'vite-plugin-testing';

export default defineConfig({
  plugins: [react(), testing()],
  test: {
    environment: 'happy-dom',
  },
});
Step 3: Create a Test File

Create a new file with the extension .test.js or .test.jsx in the tests directory. For example: src/tests/MyComponent.test.js.

Step 4: Import React Testing Library and the Component to Test

JS
import { render, screen } from '@testing-library/react';
import MyComponent from '../src/MyComponent';
Step 5: Write Test Cases

Follow these steps to write test cases:

Render the component under test.
Query the DOM for elements and assert their existence, values, or behavior.
Use methods like fireEvent to simulate user interactions.
Example Test Case:

JS
test('renders a header with the provided text', () => {
  render(<MyComponent text="Hello World" />);
  expect(screen.getByText('Hello World')).toBeInTheDocument();
});
Step 6: Run Tests

To run tests, execute the following command from your terminal:

TEXT
npm run test
Tips:

Use descriptive test names to clearly indicate what each test is testing.
Write isolated tests that focus on testing one specific aspect of your component.
Avoid mocking components unless necessary.
Consider using utilities like act from react-testing-library to manage DOM updates.
Debug failed tests by using the console output or breakpoint debugging.