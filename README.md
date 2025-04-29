# Regex-to-DFA-convertor
This repository contains a convertor from Regex to Postfixed natation, then to a lambda-NFA, further to an DFA.

# Regex -> Postfixed notation
The conversion was made using the Shunting-Yard algorithm.

First part was adding the concatenator operator "." to the regex using the "add_concat()" function:
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

Lastly, the "to_postfix_form()" function transformed the string into the postfixed form.
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
