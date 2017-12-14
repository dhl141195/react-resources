# Table of Contents

- [React Components, Elements and Instances](#react-components-elements-and-instances)
- [Scheduling](#scheduling)
- [React Fiber Architecture](#react-fiber-architecture)
- [A look inside React Fiber — how work will get done](#a-look-inside-react-fiber-%E2%80%94-how-work-will-get-done)

# React Components, Elements and Instances

## Source

<https://reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html>

## Summary

### Element

Element là plain object mà React dùng để render UI

Element có thể lồng nhau thông qua props

Có 2 loại element là DOM element và component element

Element is immutable and cheap to create

Ví dụ Element Button:

```js
{
  type: Button, // component element
  props: {
    color: 'blue',
    children: 'OK!'
  }
}
```

### Component

> Components Encapsulate Element Trees

Component nhận vào **props** và trả về **Element Tree**

```js
const DeleteAccount = (props) => ({
  type: 'div',
  props: {
    children: [{
      type: 'p',
      props: {
        children: props.title
      }
    }, {
      type: DangerButton,
      props: {
        children: 'Yep'
      }
    }, {
      type: Button,
      props: {
        color: 'blue',
        children: 'Cancel'
      }
   }]
});
```

Hoặc bằng JSX

```js
const DeleteAccount = (props) => (
  <div>
    <p>{props.title}</p>
    <DangerButton>Yep</DangerButton>
    <Button color='blue'>Cancel</Button>
  </div>
);
```

Có 2 loại component: Functional component và Class component

### Instance

> An instance is what you refer to as **this** in the component class you write. It is useful for storing local state and reacting to the lifecycle events

Chỉ có class component mới có instance

React sẽ tự động tạo instance khi gặp element với type là class element

### Top-Down Reconciliation

Khi render:

```js
ReactDOM.render({
  type: Form,
  props: {
    isSubmitted: false,
    buttonText: 'OK!'
  }
}, document.getElementById('root'));
```

React sẽ *traverse* từ trên xuống từng component cho đến khi có được 1 element tree hoàn toàn bằng DOM element (Virtual DOM)

```js
// React: You told me this...
{
  type: Form,
  props: {
    isSubmitted: false,
    buttonText: 'OK!'
  }
}

// React: ...And Form told me this...
{
  type: Button,
  props: {
    children: 'OK!',
    color: 'blue'
  }
}

// React: ...and Button told me this! I guess I'm done.
{
  type: 'button',
  props: {
    className: 'button button-blue',
    children: {
      type: 'b',
      props: {
        children: 'OK!'
      }
    }
  }
}
```

Quá trình này bắt đầu mỗi khi gọi ```ReactDOM.render()``` hoặc trigger render component bằng ```setState()```. Kết quả là 1 DOM tree (Virtual DOM). Các **renderer** như **react-dom** hoặc **react-native** sẽ dùng Virtual DOM này để render UI, so sánh và chỉ cập nhật những thay đổi cần thiết vào DOM thật

Note: Nếu ```shouldComponentUpdate()``` return false thì React sẽ bỏ qua component đó trong quá trình Reconciliation và giữ nguyên phần DOM của component so với lần render trước

# Reconciliation

## Source

<https://reactjs.org/docs/reconciliation.html>

## Summary

### Definition

> The algorithm React uses to diff one tree with another to determine which parts need to be changed.

### The Diffing Algorithm

- Nếu element khác type, toàn bộ tree tính từ element đó sẽ được rebuild (unmount, destroy old DOM nodes -> mount, insert new DOM nodes)
- Nếu **DOM element** cùng type, chỉ cập nhật lại các attribute thay đổi (lặp lại đối với tất cả các element con)
- Nếu **component element** cùng type, cập nhật lại props của các component con. Sau đó lần lượt ```componentWillReceiveProps()```,```componentWillUpdate()```, ```render()``` sẽ được gọi ở các component con

### Key on list children

React sẽ đồng thời lặp 2 list children cũ và mới để so sánh sự thay đổi của các element cùng index trong 2 list. Vì vậy trong trường hợp insert 1 element vào đầu list sẽ làm cả list bị cập nhật lại

> Truyền key cho mỗi element con. Thay vì so sánh theo từng index, React sẽ so sánh theo từng key

# Scheduling

## Source

<https://reactjs.org/docs/design-principles.html#scheduling>

## Summary

> React sticks to the “pull” approach where computations can be delayed until necessary

Chính vì vậy mà ```setState()``` là **async** -> scheduling an update

Update có nhiều mức độ ưu tiên khác nhau. Ví dụ animation update sẽ có độ ưu tiên cao hơn update dựa trên data mới fetch từ API về

Component được khai báo dưới dạng function hoặc class. React chịu trách nhiệm gọi function và khởi tạo instance. Nhờ vậy React có thể delay update, schedule, split work in chunks và update theo độ ưu tiên

# React Fiber Architecture

## Source

<https://github.com/acdlite/react-fiber-architecture>

## Summary

### Why need Fiber

> The crux of the change is transitioning from processing updates in a synchronous, recursive way to relying on asynchronous scheduling and prioritization of interface changes

Như đã đề cập trong phần [Scheduling](#scheduling), React được thiết kế để có thể delay update, schedule, split work in chunks và update theo độ ưu tiên. Nhưng hiện tại reconciler của React chưa tận dụng được những ưu điểm này, mọi update vẫn được re-render synchronous ngay lập tức. Vì vậy React được viết lại để **tận dụng khả năng scheduling** -> React Fiber

### What is Fiber

Mục tiêu của React Fiber:

- Pause work and come back to it later.
- Assign priority to different types of work.
- Reuse previously completed work.
- Abort work if it's no longer needed.

Để làm được như vậy cần tách **work** thành **unit of work** -> fiber nghĩa là **an unit of work**

React component về bản chất là [function of data](https://github.com/reactjs/react-basic#transformation). Khi render các function này sẽ được gọi. Function của component cha sẽ gọi function của các component con... Các function này sẽ được push vào **call stack**. Và khi thực hiện nhiều việc cùng lúc, call stack này lớn, sẽ dẫn đến drop frame

Sử dụng call stack không mang lại hiệu năng cao. Đặc biệt trong nhiều trường hợp, các update trước đã bị thay thế bởi update mới nhất nhưng call stack vẫn tiếp tục được thực thi cho tới khi call stack rỗng

> Solution: Fiber is reimplementation of the stack, specialized for React components. You can think of a single fiber as a **virtual stack frame**

> The advantage of reimplementing the stack is that you can keep stack frames in memory and execute them however (and whenever) you want. This is crucial for accomplishing the goals we have for scheduling.

### Structure of a fiber

> In concrete terms, a fiber is a JavaScript object that contains information about a component, its input, and its output

[List main fields of a fiber](https://github.com/acdlite/react-fiber-architecture#structure-of-a-fiber)

# A look inside React Fiber — how work will get done

## Source

Deep look: <http://makersden.io/blog/look-inside-fiber/>

Cartoon intro: <https://www.youtube.com/watch?v=ZCuYPiUIONs&t=916s>