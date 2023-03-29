## 1. Focus on Business Logic:

Our primary goal is to test the business logic of our application. This includes testing specific system parts such as RTK slices, Redux action-reducer pairs, **non-trivial** selectors, adapters, helpers, and utilities crucial to the application's functionality.

Conversely, trivial selectors and other simple functions may not require testing. For example, If a selector retrieves the current user's information from the state, such as `getCurrentUser(state)`, it may not require testing as it only returns data directly from the state without any additional processing.

```tsx
// Redux state
const state = {
  user: { id: 1, name: 'John Doe', age: 30, email: 'john@example.com' },
  posts: [
    { id: 1, title: 'Post 1', content: 'Content 1' },
    { id: 2, title: 'Post 2', content: 'Content 2' },
    { id: 3, title: 'Post 3', content: 'Content 3' },
  ],
};

// selectors.js
export const getCurrentUser = (state) => state.user;
export const getPostsByUserId = (state, userId) => state.posts.filter((post) => post.userId === userId);

// selectors.test.js
import { getCurrentUser, getPostsByUserId } from './selectors';

describe('selectors', () => {

  /* Bad */

  it('should return the current user', () => {
    const expected = {
      id: 1,
      name: 'John Doe',
      age: 30,
      email: 'john@example.com',
    };

    expect(getCurrentUser(state)).toEqual(expected);
  });

  /* Good */

  it('should return posts by user ID', () => {
    const expected = [
      { id: 1, title: 'Post 1', content: 'Content 1' },
      { id: 3, title: 'Post 3', content: 'Content 3' },
    ];

    expect(getPostsByUserId(state, 1)).toEqual(expected);
  });
});
```

## 2. Test Behavior, Not Implementation:

When writing tests, we should focus on the expected behavior of our code rather than the specific implementation details.

### hooks

If we create custom hooks that encapsulate complex logic or side effects, we should test that the hooks produce the expected outputs and effects, rather than verifying the specific internal function calls or implementation details.

```tsx
// useFetchData.js
import { useState, useEffect } from 'react';

const useFetchData = (url) => {
  const [data, setData] = useState(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchData = async () => {
      try {
        const response = await fetch(url);
        const data = await response.json();
        setData(data);
        setIsLoading(false);
      } catch (err) {
        setError(err.message);
        setIsLoading(false);
      }
    };

    fetchData();
  }, [url]);

  return { data, isLoading, error };
};

export default useFetchData;

// useFetchData.test.js
import { renderHook, act } from '@testing-library/react-hooks';
import useFetchData from './useFetchData';

const mockData = { key: 'value' };

global.fetch = jest.fn(() =>
  Promise.resolve({
    json: () => Promise.resolve(mockData),
  }),
);

describe('useFetchData', () => {
  beforeEach(() => {
    fetch.mockClear();
  });

  /* Bad */

  it('should call fetch with the correct URL', async () => {
    const url = 'https://api.example.com/data';
    renderHook(() => useFetchData(url));

    expect(fetch).toHaveBeenCalledTimes(1);
    expect(fetch).toHaveBeenCalledWith(url);
  });

  /* Good */

  it('should handle successful data fetching', async () => {
    const url = 'https://api.example.com/data';
    const { result, waitForNextUpdate } = renderHook(() => useFetchData(url));

    await waitForNextUpdate();

    expect(result.current.data).toEqual(mockData);
    expect(result.current.isLoading).toBe(false);
    expect(result.current.error).toBe(null);
  });

  it('should handle errors in data fetching', async () => {
    const url = 'https://api.example.com/data';
    const errorMessage = 'Network error';

    fetch.mockImplementationOnce(() => Promise.reject(new Error(errorMessage)));

    const { result, waitForNextUpdate } = renderHook(() => useFetchData(url));

    await waitForNextUpdate();

    expect(result.current.data).toBe(null);
    expect(result.current.isLoading).toBe(false);
    expect(result.current.error).toBe(errorMessage);
  });
});
```

### useEffect

When testing components that use useEffect for various purposes, we should focus on verifying that the component behaves as expected in response to the side effects. Some examples include:

#### Data Fetching
Test that the component updates the state, displays a loading indicator, or handles errors as expected when data is fetched.

```tsx
// DataFetchingComponent.js
import React, { useState, useEffect } from 'react';

const DataFetchingComponent = ({ url }) => {
  const [data, setData] = useState(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchData = async () => {
      try {
        const response = await fetch(url);
        const data = await response.json();
        setData(data);
        setIsLoading(false);
      } catch (err) {
        setError(err.message);
        setIsLoading(false);
      }
    };

    fetchData();
  }, [url]);

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return <div>Data: {JSON.stringify(data)}</div>;
};

export default DataFetchingComponent;

// DataFetchingComponent.test.js
import React from 'react';
import { render, waitFor } from '@testing-library/react';
import DataFetchingComponent from './DataFetchingComponent';

const mockData = { key: 'value' };

global.fetch = jest.fn(() =>
  Promise.resolve({
    json: () => Promise.resolve(mockData),
  }),
);

describe('DataFetchingComponent', () => {
  beforeEach(() => {
    fetch.mockClear();
  });

  /* Bad */

  it('should call fetch with the correct URL', () => {
    const url = 'https://api.example.com/data';
    render(<DataFetchingComponent url={url} />);

    expect(fetch).toHaveBeenCalledTimes(1);
    expect(fetch).toHaveBeenCalledWith(url);
  });

  /* Good */

  it('should display loading and data', async () => {
    const url = 'https://api.example.com/data';
    const { getByText } = render(<DataFetchingComponent url={url} />);

    expect(getByText('Loading...')).toBeInTheDocument();

    await waitFor(() => expect(getByText(`Data: ${JSON.stringify(mockData)}`)).toBeInTheDocument());
  });

  it('should handle errors', async () => {
    fetch.mockImplementationOnce(() => Promise.reject(new Error('Network error')));

    const url = 'https://api.example.com/error';
    const { getByText } = render(<DataFetchingComponent url={url} />);

    await waitFor(() => expect(getByText('Error: Network error')).toBeInTheDocument());
  });
});
```

#### Timer or Interval
Test that a component that uses `useEffect` to manage timers or intervals behaves correctly, such as triggering a state update or action at the appropriate intervals or cleaning up the timer when the component is unmounted.

```tsx
// TimerComponent.js
import React, { useState, useEffect } from 'react';

const TimerComponent = () => {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const timer = setInterval(() => {
      setCount((prevCount) => prevCount + 1);
    }, 1000);

    return () => {
      clearInterval(timer);
    };
  }, []);

  return <div>Count: {count}</div>;
};

export default TimerComponent;

// TimerComponent.test.js
import React from 'react';
import { render, act } from '@testing-library/react';
import TimerComponent from './TimerComponent';

describe('TimerComponent', () => {

  /* Bad */

  it('should call setInterval with correct delay', () => {
    const setIntervalSpy = jest.spyOn(global, 'setInterval');
    render(<TimerComponent />);

    expect(setIntervalSpy).toHaveBeenCalledTimes(1);
    expect(setIntervalSpy).toHaveBeenCalledWith(expect.any(Function), 1000);
    setIntervalSpy.mockRestore();
  });

  /* Good */

  it('should update the count and clean up the timer', async () => {
    jest.useFakeTimers();
    const { getByText } = render(<TimerComponent />);

    expect(getByText('Count: 0')).toBeInTheDocument();

    act(() => {
      jest.advanceTimersByTime(3000);
    });

    expect(getByText('Count: 3')).toBeInTheDocument();

    act(() => {
      jest.advanceTimersByTime(2000);
    });

    expect(getByText('Count: 5')).toBeInTheDocument();

    jest.useRealTimers();
  });
});
```

### adapters

Test that adapters correctly transform data between different formats. For instance, if an adapter converts API responses into a format suitable for our application's state, we should write tests to ensure that the transformed data matches the expected format.

```tsx
// apiAdapter.js
const apiAdapter = (apiResponse) => {
  // Transform API response into the desired format for application state
  const transformedData = { ... };

  return transformedData;
};

export default apiAdapter;

// apiAdapter.test.js
import apiAdapter from './apiAdapter';

describe('apiAdapter', () => {

  /* Bad */

  it('should call specific transformation functions', () => {
    const mockApiResponse = { ... };
    const mockTransformFn = jest.fn();
    apiAdapter(mockApiResponse, mockTransformFn);

    expect(mockTransformFn).toHaveBeenCalledTimes(1);
    expect(mockTransformFn).toHaveBeenCalledWith(mockApiResponse);
  });

  /* Good */

  it('should correctly transform API response into the desired format', () => {
    const mockApiResponse = { ... };
    const expectedResult = { ... };

    const transformedData = apiAdapter(mockApiResponse);

    expect(transformedData).toEqual(expectedResult);
  });
});
```

### error handling

Ensure that components and functions gracefully handle error conditions and edge cases, such as empty or invalid inputs, by verifying that the correct error messages are displayed or the appropriate fallback behavior is executed.

```ts
// InputValidationComponent.js
import React, { useState } from 'react';

const InputValidationComponent = () => {
  const [inputValue, setInputValue] = useState('');
  const [errorMessage, setErrorMessage] = useState(null);

  const validateInput = (value) => {
    if (!value) {
      setErrorMessage('Input is empty');
    } else if (!/^\d+$/.test(value)) {
      setErrorMessage('Input should only contain digits');
    } else {
      setErrorMessage(null);
    }
  };

  const handleChange = (event) => {
    setInputValue(event.target.value);
    validateInput(event.target.value);
  };

  return (
    <div>
      <input type="text" value={inputValue} onChange={handleChange} />
      {errorMessage && <div>{errorMessage}</div>}
    </div>
  );
};

export default InputValidationComponent;

// InputValidationComponent.test.js
import React from 'react';
import { render, fireEvent } from '@testing-library/react';
import InputValidationComponent from './InputValidationComponent';

describe('InputValidationComponent', () => {

  /* Bad */

  it('should call validateInput function with the input value', () => {
    const { getByRole } = render(<InputValidationComponent />);
    const input = getByRole('textbox');

    const validateInputSpy = jest.spyOn(InputValidationComponent.prototype, 'validateInput');

    fireEvent.change(input, { target: { value: '123' } });

    expect(validateInputSpy).toHaveBeenCalledTimes(1);
    expect(validateInputSpy).toHaveBeenCalledWith('123');

    validateInputSpy.mockRestore();
  });

  /* Good */

  it('should display an error message for empty input', () => {
    const { getByRole, getByText } = render(<InputValidationComponent />);
    const input = getByRole('textbox');

    fireEvent.change(input, { target: { value: '' } });

    expect(getByText('Input is empty')).toBeInTheDocument();
  });

  it('should display an error message for invalid input', () => {
    const { getByRole, getByText } = render(<InputValidationComponent />);
    const input = getByRole('textbox');

    fireEvent.change(input, { target: { value: 'abc' } });

    expect(getByText('Input should only contain digits')).toBeInTheDocument();
  });

  it('should not display any error message for valid input', () => {
    const { getByRole, queryByText } = render(<InputValidationComponent />);
    const input = getByRole('textbox');

    fireEvent.change(input, { target: { value: '123' } });

    expect(queryByText('Input is empty')).not.toBeInTheDocument();
    expect(queryByText('Input should only contain digits')).not.toBeInTheDocument();
  });
});
```

