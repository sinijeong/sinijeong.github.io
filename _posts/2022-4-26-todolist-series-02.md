---
layout: post
title: E2. 투두리스트 개발기 - 기능 구현
---

본격적으로 투두리스트 컴포넌트 제작을 해보겠습니다. 투두리스트의 가장 기본적인 기능은 무엇일까요? 아마도 할 일을 추가하고, 삭제하고, 완료한 뒤 체크를 하는 것으로 생각합니다.

위와 같은 기능을 구현하기 위해선 투두리스트의 상태 정보를 어느 한 곳에 저장하고 그 정보를 가져와서 사용해야 합니다. 리액트에서는 부모 컴포넌트에서 자식 컴포넌트에로만 값을 전달할 수 있습니다. 반대의 경우는 불가능합니다. 프로젝트의 규모가 작을 때는 괜찮겠지만, 큰 규모의 프로젝트에서는 코드 구조가 복잡해집니다.

<br/>

> 부모 컴포넌트에서 자식 컴포넌트로, 자식 컴포넌트에서 손주 컴포넌트로, 손주 컴포넌트에서 증손주 컴포넌트로... 점점 복잡해진다 (-﹏-；;)

<br/>

상태 관리 라이브러리는 상태를 글로벌하게 공유해서 위치에 얽매이지 않고 상태에 접근할 수 있게 도와주는 녀석입니다. 상태 관리 라이브러리로 Redux, Recoil, MobX 등이 있지만 이번 프로젝트에서는 jotai를 사용하기로 했습니다.

<br/>

## jotai를 사용하는 이유

1. 기타 상태관리 라이브러리에 비해 가볍다.
2. Redux는 state를 변경하려면 Action을 만들고 이를 Reducer로 전달하는 등의 사전코드 작성이 필요합니다. jotai에선 atom이라는 개별 상태를 정의하고 useAtom hook을 사용해서 그 값을 얻어올 수 있어서 형식 면에서 비교적 자유롭습니다.

<br/>

## 구현해야 할 기능

1. 사용자가 input 창에 할 일을 입력하고 추가 버튼을 클릭하면 할 일을 추가한다.
2. 사용자가 checkbox를 클릭하면 toggle이 발생한다.
3. 사용자가 todo를 왼쪽으로 드래그하면 삭제 버튼을 보여준다.
4. 사용자가 삭제 버튼을 클릭하면 할 일이 삭제된다.

<br/>

## 투두리스트 구현



### 1. 초기 투두리스트 설계

<br/>

plugins/todo/src/index.ts
{% highlight js %}
export * from './types';
export * from './hooks';

export { default } from './components/TodoList/TodoList';
export { default as TodoListForm } from './components/TodoListForm/TodoListForm';
{% endhighlight %}

<br/>

plugins/todo/src/types.ts
{% highlight ts %}
export type Todo = {
  id: string;
  title: string;
  checked: boolean;
};

export namespace Instance {
  export type GetTodos = () => Todo[];
  export type SetTodos = (payload: Todo[]) => void;
  export type AddTodo = (payload: Todo) => void;
  export type RemoveTodo = (id: string) => void;
  export type ToggleTodo = (id: string) => void;
}

export type TodoListInstance = {
  getTodos: Instance.GetTodos;
  setTodos: Instance.SetTodos;
  addTodo: Instance.AddTodo;
  removeTodo: Instance.RemoveTodo;
  toggleTodo: Instance.ToggleTodo;
};
{% endhighlight %}
todoList와 관련된 타입값

<br/>

plugins/todo/src/hooks/useTodoList.ts
{% highlight ts %}
import React from 'react';
import { useAtom } from 'jotai';
import { TodoListInstance, Instance, Todo } from '../types';
import { todosAtom } from '../atom/todosAtom';

export default function useTodoList(): TodoListInstance {
  const [todos, _setTodos] = useAtom(todosAtom);

  const getTodos = React.useCallback<Instance.GetTodos>(() => {
    return todos;
  }, [todos]);

  const setTodos = React.useCallback<Instance.SetTodos>((todos: Todo[]) => {
    _setTodos(todos);
    return;
  }, []);

  const addTodo = React.useCallback<Instance.AddTodo>(
    (todo: Todo) => {
      return setTodos([todo, ...todos]);
    },
    [todos]
  );

  const removeTodo = React.useCallback<Instance.RemoveTodo>(
    (id: string) => {
      return setTodos(todos.filter((todo) => todo.id !== id));
    },
    [todos]
  );

  const toggleTodo = React.useCallback<Instance.ToggleTodo>(
    (id: string) => {
      return setTodos(
        todos.map((todo) =>
          todo.id !== id
            ? todo
            : {
                ...todo,
                checked: !todo.checked,
              }
        )
      );
    },
    [todos]
  );

  return {
    getTodos,
    setTodos,
    addTodo,
    removeTodo,
    toggleTodo,
  };
}
{% endhighlight %}
todoList와 관련된 상태값을 외부에서 참조 및 변경할 수 있는 React Hook

<br/>

plugins/todo/src/atom/todosAtom.ts
{% highlight ts %}
import { atom } from 'jotai';
import { Todo } from '../types';

export const todosAtom = atom<Todo[]>([]);
{% endhighlight %}
Todo 배열을 속성으로 갖는 atom을 정의합니다.

<br/>

plugins/todo/src/components/TodoList/TodoList.tsx
{% highlight tsx %}
import React, { FC } from 'react';
import { Todo } from '../../types';
import { useAtom } from 'jotai';
import { todosAtom } from '../../atom/todosAtom';
import '../../style.css';

type TodoListProps = {
  initialTodos: Todo[];
  CardComponent?: React.ElementType<{ todo: Todo }>;
};

function DefaultCard({ todo }: { todo: Todo }) {
  return <div>{todo.title}</div>;
}

const TodoList: FC<TodoListProps> = ({
  initialTodos,
  CardComponent = DefaultCard,
}) => {
  const [todos, setTodos] = useAtom(todosAtom);

  React.useEffect(() => {
    setTodos(initialTodos);
  }, []);

  return (
    <>
      <ul className='todolist'>
        {todos.map((todo) => (
          <CardComponent todo={todo} />
        ))}
      </ul>
    </>
  );
};

<br/>

export default TodoList;
{% endhighlight %}
현재 Todo 배열 값을 Todo컴포넌트로 렌더링하는 역할을 합니다. useEffect를 사용하여 마운트시에 initialTodos를 todosAtom에 정의합니다. jotai를 사용하여 useState 대신 useAtom을 사용하여 state를 변경할 수 있습니다. 또한 CardComponent를 props로 넘겨주어 커스텀 카드로 사용할 수 있게 해줬습니다.

<br/>

plugins/todo/src/components/TodoForm/TodoForm.tsx
{% highlight tsx %}
import React, { FC } from 'react';
import { v4 } from 'uuid';
import { Todo } from '../../types';

type TodoListFormProps = {
  onAddTodo?: (todo: Todo) => void | Promise<void>;
};

const TodoListForm: FC<TodoListFormProps> = ({ onAddTodo }) => {
  const inputRef = React.useRef<HTMLInputElement>(null);

  const onAddTodoHandler = (event: React.FormEvent) => {
    event.preventDefault();
    if (onAddTodo) {
      if (inputRef.current) {
        onAddTodo({
          id: v4(),
          title: inputRef.current.value,
          checked: false,
        });
        inputRef.current.value = '';
      }
    }
  };

  return (
    <form className='todolist__form' onSubmit={onAddTodoHandler}>
      <input placeholder='할 일을 입력하세요' ref={inputRef} />
      <button type='submit'>추가</button>
    </form>
  );
};

export default TodoListForm;
{% endhighlight %}
투두 리스트의 입력폼 컴포넌트입니다.

<br/>

## 2. CheckBox

할 일을 완료하고 나면 체크도 할 수 있어야겠죠. 할 일의 완료 여부를 표시하는 checkBox를 만들어보겠습니다. 

plugins/todo/src/index.ts
{% highlight ts %}
...
export { default as CheckBox } from './components/CheckBox/CheckBox';
export { default as CheckProvider } from './components/CheckBox/CheckProvider';
{% endhighlight %}

<br/>

plugins/todo/src/components/CheckBox/CheckBox.tsx
{% highlight tsx %}
import React, { FC } from 'react';
import { useCheck } from './CheckProvider';

type Props = {
  Component?: React.FunctionComponent<React.HTMLProps<HTMLInputElement>>;
  onCheck?: React.ChangeEventHandler<HTMLInputElement>;
};

const CheckBox: FC<Props> = ({
  Component = (props) => <input {...props} />,
  onCheck,
}) => {
  const checked = useCheck();
  return <Component type='checkbox' checked={checked} onChange={onCheck} />;
};

export default CheckBox;
{% endhighlight %}
CheckBox또한 커스텀 컴포넌트를 사용할 수 있도록  props에 Component속성을 추가해줬습니다.

<br/>

plugins/todo/src/components/CheckBox/CheckProvider.tsx
{% highlight tsx %}
import React, { FC } from 'react';

const Context = React.createContext<boolean | undefined>(undefined);

type Props = {
  checked: boolean;
};

const CheckProvider: FC<Props> = ({ checked, children }) => {
  return <Context.Provider value={checked}>{children}</Context.Provider>;
};

export const useCheck = () => {
  const checked = React.useContext(Context);
  if (checked === undefined) {
    throw new Error('No CheckProvider is available in react context');
  }
  return checked;
};

export default CheckProvider;
{% endhighlight %}
Context: 체크 여부를 포함하는 컨텍스트

CheckProvider: 체크 여부를 props로 받아서 Context에 제공하는 역할

useCheck: Context의 값을 받아오는 React Hook

<br/>

다음은 CheckProvider와 CheckBox의 사용예시입니다. 

{% highlight tsx %}
<CheckProvider checked={...}>
	<CheckBox Component={CheckBoxA} onCheck={...}/>
	<p>example</p>
</CheckProvider>
{% endhighlight %}

![Success Screen]({{ site.baseurl }}/assets/img/todolist_series_2/checkbox_1.gif)

<br/>

CheckProvider안이라면 복수의 CheckBox가 오더라도 동일한 checked값에 영향을 받습니다.

{% highlight tsx %}
<CheckProvider checked={...}>
	<CheckBox Component={CheckBoxA} onCheck={...}/>
	<CheckBox Component={CheckBoxA} onCheck={...}/>
	<p>example</p>
	<CheckBox Component={CheckBoxA} onCheck={...}/>
	<CheckBox Component={CheckBoxA} onCheck={...}/>
</CheckProvider>
{% endhighlight %}

![Success Screen]({{ site.baseurl }}/assets/img/todolist_series_2/checkbox_2.gif)

## 3. Slide-to-Delete 카드 제작
이제 커스텀 카드인 SlideToDeleteCard를 제작해서 TodoList에 CardComponent로 넘겨주도록 하겠습니다.

#### 구현
1. 초기엔 deleteButton이 카드에 가려서 보이지 않는다.
2. 카드를 왼쪽으로 드래그하면 카드 버튼이 보인다.
3. deleteButton을 누르면 해당 카드가 삭제된다.

<br/>

packages/app/src/cards/SlideToDeleteCard.tsx
{% highlight tsx %}
import React from 'react';
import { Todo, CheckBox, useTodoList } from '@todolist/plugin-todo';
import { BiTrash } from 'react-icons/bi';
import { useSpring, animated } from '@react-spring/web';
import { useDrag } from '@use-gesture/react';
import styled, { css } from 'styled-components';

export type Props = {
  todo: Todo;
  draggableMaxX?: number;
};

const SlideToDeleteCard = ({ todo, draggableMaxX = 42 }: Props) => {
  const { removeTodo, toggleTodo } = useTodoList();
  const isDragging = React.useRef(false);
  const dragged = React.useRef(false);

  const onRemoveTodoHandler = React.useCallback(async () => {
    if (removeTodo) {
      await removeTodo(todo.id);
    }
  }, [todo.id, removeTodo]);

  const onToggleTodoHandler = React.useCallback(
    async (event: React.ChangeEvent) => {
      event.stopPropagation();
      if (toggleTodo) {
        await toggleTodo(todo.id);
      }
    },
    [todo.id, toggleTodo]
  );

  const [{ x, y }, api] = useSpring(() => ({ x: 0, y: 0 }));

  const bind = useDrag(({ down, direction, movement: [mx] }) => {
    api.start(() => {
      if (direction[0] === -1) {
        if (mx <= -draggableMaxX) {
          if (down) {
            if (!dragged.current) {
              dragged.current = true;
            }
          }
          return { x: -draggableMaxX };
        }
        return { x: down ? mx : 0 };
      } else if (direction[0] === 1) {
        if (dragged.current) {
          if (down) {
            dragged.current = false;
            return { x: 0 };
          }
          return { x: mx };
        }
      }
    });
  });

  return (
    <li>
      <DeleteCard onClick={onRemoveTodoHandler}>
        <span>
          <BiTrash />
        </span>
      </DeleteCard>
      <animated.div
        {...(bind as any)()}
        style={{
          x,
          y,
          touchAction: 'none',
          height: '50px',
        }}
      >
        <Card checked={todo.checked}>
          <CheckBox onCheck={onToggleTodoHandler} />
          <p>{todo.title}</p>
        </Card>
      </animated.div>
    </li>
  );
};

export default SlideToDeleteCard;

const Card = styled.div<{ checked: boolean }>`
	...
`;

const DeleteCard = styled.div`
	...
`;
{% endhighlight %}

<br/>

packages/app/src/pages/Home/Home.tsx
{% highlight tsx %}
...
import SlideToDeleteCard from '../../cards/SlideToDeleteCard';

const HomePage = () => {
	return (
		...
			<TodoList initialTodos={todos} CardComponent={SlideToDeleteCard} />
		...
	)
}
{% endhighlight %}

<br/>

![Success Screen]({{ site.baseurl }}/assets/img/todolist_series_2/checkbox_3.gif)

실행 화면은 다음과 같다.

<br/>

#### ⚠️ 이벤트 중복 발생 문제

![Success Screen]({{ site.baseurl }}/assets/img/todolist_series_2/checkbox_4.gif)

그러나 한 가지 신경쓰이는 부분이 있다. 체크박스위에서 드래그 할 때 MouseUp 이벤트가 발생하면 체크박스 또한 토글 된다는 점이다. 문제점을 해결하기 위해서 드래깅이 진행 중일때는 체크박스에서 MouseUp 이벤트가 발생해도 무시해야 한다.

packages/app/src/cards/SlideToDeleteCard.tsx
{% highlight tsx %}
import React from 'react';
import { Todo, CheckBox, useTodoList } from '@todolist/plugin-todo';
import { BiTrash } from 'react-icons/bi';
import { useSpring, animated } from '@react-spring/web';
import { useDrag } from '@use-gesture/react';
import styled, { css } from 'styled-components';

export type Props = {
  todo: Todo;
  draggableMaxX?: number;
};

const SlideToDeleteCard = ({ todo, draggableMaxX = 42 }: Props) => {
  ...
  const isDragging = React.useRef(false);
  const dragged = React.useRef(false);

	...

	const onToggleTodoHandler = React.useCallback(
    async (event: React.ChangeEvent) => {
      event.stopPropagation();
      if (toggleTodo) {
				// 드래그 중이 아니면 토글을 한다.
        if (!isDragging.current) {
          await toggleTodo(todo.id);
        }
      }
    },
    [todo.id, toggleTodo]
  );

	...

  const bind = useDrag(({ down, direction, movement: [mx] }) => {
    if (!isDragging.current) {
			// 드래깅 시작
      const trigger = mx < -0.2 || mx > 0.2;
      if (trigger) {
        isDragging.current = true;
      }
    }

    api.start(() => {
      ...
    });

    if (!down) {
			// 드래깅 완료
      if (isDragging.current) {
        setTimeout(() => {
          isDragging.current = false;
        }, 0);
      }
    }
  });

  return (
    ...
  );
};

export default SlideToDeleteCard;
{% endhighlight %}
이벤트 중복 발생을 방지하기 위해 isDragging, dragged 변수를 만들었다. 드래깅이 발생하면 trigger 플래그가 참이 되며 isDragging값이 참이 된다. MouseUp이벤트가 발생했을 때 체크박스 click과 드래깅 완료 이벤트가 발생하는데 체크박스 click의 경우 isDragging이 true 일 경우 toggle을 발생시키지 않는다. 드래깅 완료 이벤트에선 isDragging이 참이라면 isDragging을 false로 바꾼다.

<br>

#### 클릭 이벤트 루프

카드 드래그 이벤트 루프 과정은 다음과 같다.

1. 호출 스택에 쌓임
![Success Screen]({{ site.baseurl }}/assets/img/todolist_series_2/event_loop_1.png)

2. 드래깅 완료 이벤트가 실행되면서 호출 스택에서 제거되고 SetTimeout이 다시 호출 스택에 쌓임
![Success Screen]({{ site.baseurl }}/assets/img/todolist_series_2/event_loop_2.png)

3. CheckBox 클릭 이벤트가 실행되면서 호출 스택에서 제거됨
![Success Screen]({{ site.baseurl }}/assets/img/todolist_series_2/event_loop_3.png)

4. SetTimeout이 실행되면서 cb이 백그라운드로 이동함
![Success Screen]({{ site.baseurl }}/assets/img/todolist_series_2/event_loop_4.png)

5. 0초후 백그라운드에서 태스트 큐로 cb 이동
![Success Screen]({{ site.baseurl }}/assets/img/todolist_series_2/event_loop_5.png)

6. 이벤트 루프가 태스크 큐의 cb을 다시 호출 스택에 쌓음
![Success Screen]({{ site.baseurl }}/assets/img/todolist_series_2/event_loop_6.png)

<br/>

#### 실행화면

![Success Screen]({{ site.baseurl }}/assets/img/todolist_series_2/checkbox_5.gif)

잘 작동합니다😄

<br/>

### Series: 투두리스트 개발기 

[E1. 투두리스트 개발기 - Mono-Repo구축(w.npm workspaces + carco)](/2022/04/24/todolist-series-01.html)<br>
***[E2. 투두리스트 개발기 - 기능 구현](/2022/04/26/todolist-series-02.html)***