#  Regex to DFA Converter

This project implements a full pipeline that converts **regular expressions** into **Deterministic Finite Automata (DFA)** using classical algorithms from automata theory.

## 📌 Overview

The conversion process consists of the following stages:

1. **Regex → Postfix Notation**
   - Implemented using the **Shunting Yard algorithm**.
   - Supports standard regex operators: `|` (union), `.` (concatenation), `*` (Kleene star), `+` (one or more), and `?` (zero or one).

  The conversion is done by the "add_concat()" function,
  ```
def add_concat(regex):
    result = ""
    for i in range(len(regex)):
        char = regex[i]
        result += char
        if i + 1 < len(regex):
            if ( char.isalnum() or char in [")", "*", "?", "+"]) and ( regex[i+1].isalnum() or regex[i+1] == "("):
                result += "."

    return result
  ```
 followed by the "to_postfix_form()":
 ```
def to_postfix_form(regex):
    result = ''
    stack = deque()

    for char in regex:
        if char.isalnum():
            result += char
            continue
        if char == "(":
            stack.append(char)
            continue
        if char == ")":
            while len(stack) > 0 and stack[-1] != "(":
                result += stack.pop()
            stack.pop()
            continue
        if isOperator(char):
            while len(stack) > 0 and isOperator(stack[-1]) and (precedence(char) < precedence(stack[-1]) or ( precedence(char) == precedence(stack[-1]) and char not in ["?", "*", "+"]) ):
                result += stack.pop()
            stack.append(char)
            continue

    while len(stack) > 0:
        result += stack.pop()

    return result      
```


2. **Postfix → λ-NFA (Epsilon-NFA)**
   - Built using **Thompson's construction algorithm**.
   - Constructs a Non-deterministic Finite Automaton with lambda (ε) transitions.
  
    ```
    def build_NFA(regex):
    state_number = [0]
    nfa_stack = deque()

    def new_state():
        state = state_number[0]
        state_number[0] += 1
        return f"q{state}"
    
    def merge(d1,d2):
        for key, value in d2.items():
            if key in d1:
                d1[key] += value
            else:
                d1[key] = value

        return d1
    
    for char in regex:
        if char.isalnum():
            start = new_state()
            final_states = new_state()
            foo = { start: {char: [final_states], None: []},
                   final_states: {None: []},
                   }
            nfa = { 
                "q0" : start,
                "final": final_states,
                "foo": foo
            }
            
            nfa_stack.append(nfa)
            continue

        if char == "*":
            nfa = nfa_stack.pop()
            start = new_state()
            final_states = new_state()

            foo = { start: {None: [final_states, nfa["q0"]]},
                    final_states: {None: [start]},
                    }
            
            nfa["foo"][nfa["final"]][None] += [final_states]
            
            nfa["foo"] = merge(nfa["foo"], foo)

            new_nfa = {
                "q0" : start,
                "final": final_states,
                "foo": nfa["foo"]
            }
               
            nfa_stack.append(new_nfa)
            continue

        if char == "+":
            nfa = nfa_stack.pop()
            start = new_state()
            final_states = new_state()

            foo = { final_states: {None: [start]},
                   start: {None: [nfa["q0"]]} }
            
            nfa["foo"][nfa["final"]][None] += [final_states]
            
            nfa["foo"] = merge(nfa["foo"], foo)

            new_nfa = {
                "q0" : start,
                "final": final_states,
                "foo": nfa["foo"]
            }
            nfa_stack.append(new_nfa)
            continue

        if char == "?":
            nfa = nfa_stack.pop()
            start = new_state()
            final_states = new_state()

            foo = { start: {None: [final_states ,nfa["q0"]]}, 
                   final_states: {None: []}
                   }
            
            nfa["foo"][nfa["final"]][None] += [final_states]
            
            nfa["foo"] = merge(nfa["foo"], foo)

            new_nfa = {
                "q0" : start,
                "final": final_states,
                "foo": nfa["foo"]
            }
            nfa_stack.append(new_nfa)
            continue

        if char == "|":
            nfa1 = nfa_stack.pop()
            nfa2 = nfa_stack.pop()
            start = new_state()
            final_states = new_state()

            foo = {
                start: {None: [nfa1["q0"], nfa2["q0"]]},
                final_states: {None: []}
            }

            nfa1["foo"][nfa1["final"]][None] += [final_states]
            nfa2["foo"][nfa2["final"]][None] += [final_states]

            nfa1["foo"] = merge(nfa1["foo"], foo)
            nfa1["foo"] = merge(nfa1["foo"], nfa2["foo"])

            new_nfa = {
                "q0" : start,
                "final": final_states,
                "foo": nfa1["foo"]
            }
            nfa_stack.append(new_nfa)
            continue

        if char == ".":
            nfa_second = nfa_stack.pop()
            nfa_first = nfa_stack.pop()
            start = new_state()
            final_states = new_state()

            foo = {
                start: {None: [nfa_first["q0"]]},
                final_states: {None: []}
            }

            nfa_first["foo"][nfa_first["final"]][None] += [nfa_second["q0"]]
            nfa_second["foo"][nfa_second["final"]][None] += [final_states]

            nfa_first["foo"] = merge(nfa_first["foo"], foo)
            nfa_first["foo"] = merge(nfa_first["foo"], nfa_second["foo"])

            new_nfa = {
                "q0" : start,
                "final": final_states,
                "foo": nfa_first["foo"]
            }

            nfa_stack.append(new_nfa)
            continue

    if len(nfa_stack) != 1:
        raise Exception(f"The nfa_stack has lenght: {len(nfa_stack)}")

    return nfa_stack.pop()
    ```

3. **λ-NFA → DFA**
   - Uses **epsilon-closure** and **subset construction** to eliminate non-determinism.
  
   ```
   def NFA_to_DFA(nfa):
    Q = set()
    Sigma = set()
    for state in nfa["foo"]:
        Q.add(state)

    for state in Q:
        for char in nfa["foo"][state]:
            if char != None:
                Sigma.add(char)

    def lambda_set(states):
        result_set = set(states)
        stack = deque()
        for state in states:
            stack.append(state)

        while len(stack) > 0:
            state = stack.pop()
            for next_state in nfa["foo"][state][None]:
                if next_state not in result_set:
                    result_set.add(next_state)
                    stack.append(next_state)

        return result_set
    
    def moves_from_states(states, char):
        result_set = set()
        stack = deque(lambda_set(states))

        while len(stack) > 0:
            state = stack.pop()
            if char in nfa["foo"][state]:
                for next_state in nfa["foo"][state][char]:
                    result_set.add(next_state)

        return frozenset(lambda_set(result_set))
    
    
    init_states = frozenset(lambda_set([nfa["q0"]]))
    
    state_map = { init_states: "q0" }

    def get_name(state):
        for key, value in state_map.items():
            if value == state:
                return frozenset(key)
        return None

    foo = {}
    stack_states = deque([init_states])
    state_count = 1

    while len(stack_states) > 0:
        current_set = stack_states.pop()
        for char in Sigma:
            result_set = moves_from_states(current_set, char)
            if result_set:
                if result_set not in state_map.keys():
                    state_map[result_set] = f"q{state_count}"
                    state_count += 1
                    stack_states.append(result_set)
                if state_map[current_set] not in foo:
                    foo[state_map[current_set]] = {}
                foo[state_map[current_set]][char] = [state_map[result_set]]

                
                
    final_states = set([])
    for sets, name in state_map.items():
        if nfa["final"] in sets:
            final_states.add(name)


    dfa = {
        "q0": "q0",
        "final": final_states,
        "foo": foo
    }
            
    return dfa
   ```
