import 'package:flutter/material. dart';
import 'package: flutter/services.dart';
import 'package:flutter/foundation.dart'; // For kDebugMode
import 'dart:math' as math;

void main() {
  // 1. Critical:  Fail fast if logic is broken
  if (kDebugMode) {
    try {
      _runStrictTests();
    } catch (e) {
      // Print error and KILL execution. 
      print("FATAL LOGIC ERROR: $e");
      throw Exception("App startup aborted due to failed logic verification.");
    }
  }
  runApp(const JellyCalcApp());
}

// --- 1.  Comprehensive Regression Tests ---
void _runStrictTests() {
  print('--- Running Deep Regression Suite ---');

  void check(String expr, dynamic expected) {
    try {
      String clean = expr.replaceAll('x', '*').replaceAll('÷', '/');
      double result = Parser(clean).parse();
      
      if (expected is String) {
         throw Exception('FAILED: "$expr" -> Expected error "$expected", but got result $result');
      } else {
         if ((result - (expected as double)).abs() > 0.0001) {
            throw Exception('FAILED: "$expr" -> Expected $expected, got $result');
         }
      }
    } catch (e) {
      if (expected is String) {
        if (! e.toString().contains(expected)) {
           throw Exception('FAILED: "$expr" -> Expected error containing "$expected", got "$e"');
        }
      } else {
        throw Exception('CRASH: "$expr" -> $e');
      }
    }
  }

  // A. Valid Math
  check('2+2', 4.0);
  check('5--2', 7.0); 
  check('10%3', 1.0); 
  check('-. 5+1', 0.5); // Decimal after unary minus

  // B. Strict Error Boundaries (Fix 2, 3, 4, 9, 10)
  check('5+', 'Incomplete');       // Trailing operator
  check('. ', 'Invalid Number');    // Standalone decimal
  check('5/0', 'Div by Zero');     
  check('5%0', 'Div by Zero');     // Modulo div by zero
  check('10%3%2', 'Ambiguous');    // Chained modulo rejected
  check('5. 5%2', 'Int Mod Only'); 
  check('2*(3+2', "Missing ')'"); 
  check('+', "Unexpected");       

  print('--- All Logic Checks Passed ---');
}

// --- 2. Theme ---
class AppTheme {
  static const Color softPink = Color(0xFFFDEBF7);
  static const Color softPurple = Color(0xFFE0C3FC);
  static const Color softBlue = Color(0xFF8EC5FC);
  static const Color glassWhite = Color(0x4DFFFFFF); 
  static const Color glassBorder = Color(0x80FFFFFF); 
  static const Color btnOperator = Color(0xCCFFB7B2); 
  static const Color btnEqual = Color(0xE6B5EAD7);    
  static const Color btnAction = Color(0xCCC7CEEA);   
  static const Color btnDefault = Color(0x80FFFFFF);
  static const Color textError = Color(0xFFD32F2F);

  static const TextStyle displayInput = TextStyle(
    fontSize:  28, color: Colors.black54, fontWeight: FontWeight.w400, letterSpacing: 1.2,
  );
  static const TextStyle displayResult = TextStyle(
    fontSize: 56, color: Colors.black87, fontWeight: FontWeight.bold,
  );
  static const TextStyle displayError = TextStyle(
    fontSize: 40, color: textError, fontWeight: FontWeight. bold,
  );
}

// --- 3. Main App ---
class JellyCalcApp extends StatelessWidget {
  const JellyCalcApp({super.key});
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'JellyCalc',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(useMaterial3: true, fontFamily: 'Rounded Mplus 1c'),
      home: const CalculatorScreen(),
    );
  }
}

class CalculatorScreen extends StatefulWidget {
  const CalculatorScreen({super.key});
  @override
  State<CalculatorScreen> createState() => _CalculatorScreenState();
}

class _CalculatorScreenState extends State<CalculatorScreen> with TickerProviderStateMixin {
  static const List<String> _keypadGrid = [
    'C', '(', ')', '⌫',
    '7', '8', '9', '÷',
    '4', '5', '6', 'x',
    '1', '2', '3', '-',
    '. ', '0', '%', '+',
  ];

  String _input = '0';
  String _result = '0';
  bool _isErrorState = false; 
  bool _freshResult = false; 
  
  late AnimationController _entranceController;
  late AnimationController _resultPulseController;
  late AnimationController _shakeController;
  late Animation<double> _scaleAnimation;
  late Animation<double> _fadeAnimation;
  
  final FocusNode _focusNode = FocusNode();

  @override
  void initState() {
    super.initState();
    _entranceController = AnimationController(vsync: this, duration: const Duration(milliseconds: 800));
    _scaleAnimation = CurvedAnimation(parent: _entranceController, curve:  Curves.elasticOut);
    _fadeAnimation = CurvedAnimation(parent:  _entranceController, curve:  Curves.easeIn);
    _resultPulseController = AnimationController(vsync: this, duration: const Duration(milliseconds: 200));
    _shakeController = AnimationController(vsync: this, duration: const Duration(milliseconds: 400));
    _entranceController.forward();
    
    WidgetsBinding.instance.addPostFrameCallback((_) => _focusNode.requestFocus());
  }

  @override
  void dispose() {
    _entranceController.dispose();
    _resultPulseController.dispose();
    _shakeController.dispose();
    _focusNode.dispose();
    super.dispose();
  }

  // --- 11. Robust Keyboard (No keyLabels) ---
  void _handleKeyEvent(RawKeyEvent event) {
    if (event is RawKeyDownEvent) {
      final key = event.logicalKey;
      String? val;
      
      // Explicit Mapping for reliability
      if (key == LogicalKeyboardKey.enter || key == LogicalKeyboardKey. numpadEnter || key == LogicalKeyboardKey. equal) val = '=';
      else if (key == LogicalKeyboardKey.backspace) val = '⌫';
      else if (key == LogicalKeyboardKey. escape) val = 'C';
      else if (key == LogicalKeyboardKey.digit0 || key == LogicalKeyboardKey.numpad0) val = '0';
      else if (key == LogicalKeyboardKey.digit1 || key == LogicalKeyboardKey. numpad1) val = '1';
      else if (key == LogicalKeyboardKey.digit2 || key == LogicalKeyboardKey.numpad2) val = '2';
      else if (key == LogicalKeyboardKey. digit3 || key == LogicalKeyboardKey.numpad3) val = '3';
      else if (key == LogicalKeyboardKey.digit4 || key == LogicalKeyboardKey.numpad4) val = '4';
      else if (key == LogicalKeyboardKey.digit5 || key == LogicalKeyboardKey. numpad5) val = '5';
      else if (key == LogicalKeyboardKey.digit6 || key == LogicalKeyboardKey.numpad6) val = '6';
      else if (key == LogicalKeyboardKey. digit7 || key == LogicalKeyboardKey.numpad7) val = '7';
      else if (key == LogicalKeyboardKey.digit8 || key == LogicalKeyboardKey.numpad8) val = '8';
      else if (key == LogicalKeyboardKey.digit9 || key == LogicalKeyboardKey. numpad9) val = '9';
      else if (key == LogicalKeyboardKey.add || key == LogicalKeyboardKey. numpadAdd) val = '+';
      else if (key == LogicalKeyboardKey.minus || key == LogicalKeyboardKey.numpadSubtract) val = '-';
      else if (key == LogicalKeyboardKey. asterisk || key == LogicalKeyboardKey.numpadMultiply) val = 'x';
      else if (key == LogicalKeyboardKey. slash || key == LogicalKeyboardKey.numpadDivide) val = '÷';
      else if (key == LogicalKeyboardKey.period || key == LogicalKeyboardKey.numpadDecimal) val = '.';
      else if (key == LogicalKeyboardKey.percent) val = '%';
      else if (key == LogicalKeyboardKey.parenthesisLeft) val = '(';
      else if (key == LogicalKeyboardKey.parenthesisRight) val = ')';

      if (val != null) _onButtonPressed(val);
    }
  }

  void _onButtonPressed(String value) {
    HapticFeedback.lightImpact();
    setState(() {
      // 6. Fix Error State Logic:  Only reset if input changes validly
      if (_isErrorState) {
        if (value != 'C' && value != '⌫') {
           // If user types, we keep the input but clear the error message
           _result = '0'; 
           _isErrorState = false;
        }
      }

      // 1. Clear
      if (value == 'C') {
        _input = '0'; _result = '0'; _freshResult = false; _isErrorState = false;
        return;
      }
      
      // 2. Backspace
      if (value == '⌫') {
        if (_freshResult) {
          // 7. Edit Result instead of Clear
          _freshResult = false; 
          // Treat current result as input for editing
          if (_input.isNotEmpty) {
             _input = _input.substring(0, _input.length - 1);
          }
          if (_input.isEmpty) _input = '0';
        } else {
          if (_input.length > 1) {
            _input = _input. substring(0, _input.length - 1);
          } else {
            _input = '0';
          }
        }
        return;
      }

      // 3. Evaluate
      if (value == '=') {
        _calculateResult();
        return;
      }

      bool isOperator = ['+', '-', 'x', '÷', '%']. contains(value);

      // 4. Handle Fresh State
      if (_freshResult) {
        if (isOperator) {
           _freshResult = false; 
        } else {
           _input = ''; _freshResult = false; 
        }
      }

      // 5. Input Building
      if (_input == '0' && !isOperator && value != '. ') {
        _input = value; 
      } else {
        // A.  Prevent Leading Unary Plus
        if (_input.isEmpty && value == '+') {
          _triggerShake();
          return;
        }

        // 5. Smart Parenthesis Blocking
        if (value == ')') {
          int open = '('. allMatches(_input).length;
          int close = ')'.allMatches(_input).length;
          if (close >= open) {
             _triggerShake(); // Block invalid close
             return; 
          }
        }

        // B. Handle Decimal
        if (value == '.') {
          // 8. Fix Decimal after Unary/Paren
          if (_canAddDecimal()) {
            // Check for empty, operator, or parenthesis
            if (_input. isEmpty || ['+', '-', 'x', '÷', '(', '%'].contains(_input[_input.length - 1])) {
              _input += "0. ";
            } else {
              _input += ". ";
            }
          } else {
            _triggerShake();
          }
        } 
        // C. Handle Operators 
        else if (isOperator) {
          String last = _input.isNotEmpty ? _input[_input.length - 1] : '';
          bool lastIsOp = ['+', '-', 'x', '÷', '%'].contains(last);
          
          if (lastIsOp) {
            if (value == '-') {
              if (last != '-') _input += value;
              else _triggerShake(); 
            } else {
              _input = _input.substring(0, _input.length - 1) + value;
            }
          } else {
             _input += value;
          }
        }
        else {
          _input += value;
        }
      }
    });
  }

  bool _canAddDecimal() {
    if (_input.isEmpty) return true;
    int lastOpIndex = -1;
    for (int i = _input.length - 1; i >= 0; i--) {
      if (['+', '-', 'x', '÷', '(', ')', '%'].contains(_input[i])) {
        lastOpIndex = i;
        break;
      }
    }
    String lastSegment = _input.substring(lastOpIndex + 1);
    return ! lastSegment.contains('.');
  }

  void _triggerShake() {
    _shakeController.forward(from: 0);
    HapticFeedback.heavyImpact();
  }

  void _calculateResult() {
    try {
      String clean = _input.replaceAll('x', '*').replaceAll('÷', '/');
      if (clean.isEmpty) return;

      Parser parser = Parser(clean);
      double eval = parser.parse();

      if (eval. isInfinite) throw ParserException("Div by Zero");
      if (eval.isNaN) throw ParserException("Math Error");

      String finalResult;
      if (eval % 1 == 0) {
        finalResult = eval.toInt().toString();
      } else {
        finalResult = eval.toStringAsFixed(6);
        while (finalResult.endsWith('0')) finalResult = finalResult.substring(0, finalResult.length - 1);
        if (finalResult.endsWith('.')) finalResult = finalResult.substring(0, finalResult.length - 1);
      }

      setState(() { 
        _result = finalResult; 
        _input = finalResult; 
        _freshResult = true; 
        _isErrorState = false;
      });
      _resultPulseController.reset();
      _resultPulseController.forward();
      
    } on ParserException catch (e) {
      setState(() { 
        _result = e.message; 
        _isErrorState = true;
        _freshResult = false; 
      });
      _triggerShake();
    } catch (e) {
      setState(() { 
        _result = "Error"; 
        _isErrorState = true;
        _freshResult = false;
      });
      _triggerShake();
    }
  }

  void _closeApp() async {
    await _entranceController.reverse();
    SystemNavigator.pop();
  }

  @override
  Widget build(BuildContext context) {
    return RawKeyboardListener(
      focusNode: _focusNode,
      onKey: _handleKeyEvent,
      child:  Scaffold(
        body: Stack(
          children: [
            Container(
              decoration: const BoxDecoration(
                gradient: LinearGradient(
                  begin: Alignment.topLeft, end: Alignment.bottomRight,
                  colors: [AppTheme.softPink, AppTheme.softPurple, AppTheme.softBlue],
                ),
              ),
            ),
            Center(
              child: ScaleTransition(
                scale: _scaleAnimation,
                child: FadeTransition(
                  opacity:  _fadeAnimation,
                  child:  Padding(
                    padding: const EdgeInsets.all(20.0),
                    child: RepaintBoundary(
                      child: ClipRRect(
                        borderRadius: BorderRadius.circular(40),
                        child: Container(
                            width: double.infinity,
                            height: MediaQuery.of(context).size.height * 0.9,
                            decoration: BoxDecoration(
                              color: AppTheme.glassWhite, 
                              borderRadius: BorderRadius.circular(40),
                              border: Border.all(color: AppTheme.glassBorder),
                              boxShadow: [BoxShadow(color: Colors.black.withOpacity(0.05), blurRadius: 20, spreadRadius: 5)],
                            ),
                            child: Column(
                              children: [
                                Align(
                                  alignment:  Alignment.topRight,
                                  child:  Padding(padding: const EdgeInsets.all(16.0), child: IconButton(icon: const Icon(Icons.close_rounded, color: Colors.black54), onPressed: _closeApp)),
                                ),
                                Expanded(flex: 3, child: _buildDisplayArea()),
                                Expanded(flex: 5, child:  Padding(padding: const EdgeInsets.all(16.0), child: _buildKeypad())),
                              ],
                            ),
                          ),
                      ),
                    ),
                  ),
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildDisplayArea() {
    return AnimatedBuilder(
      animation: _shakeController,
      builder: (context, child) {
        return Transform.translate(
          offset: Offset(math.sin(_shakeController.value * math.pi * 4) * 10, 0),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.end,
            crossAxisAlignment: CrossAxisAlignment.end,
            children: [
              Padding(
                padding: const EdgeInsets.symmetric(horizontal: 24. 0),
                child: SingleChildScrollView(
                  scrollDirection: Axis.horizontal, reverse: true,
                  child:  Text(_input, style: AppTheme.displayInput),
                ),
              ),
              const SizedBox(height: 10),
              Padding(
                padding: const EdgeInsets.symmetric(horizontal: 24.0, vertical: 10),
                child: ScaleTransition(
                  scale:  Tween(begin: 1.0, end: 1.15).animate(CurvedAnimation(parent: _resultPulseController, curve:  Curves.elasticOut)),
                  child:  FittedBox(
                    fit:  BoxFit.scaleDown, 
                    child: Text(
                      _result, 
                      style: _isErrorState ? AppTheme. displayError : AppTheme.displayResult
                    )
                  ),
                ),
              ),
            ],
          ),
        );
      },
    );
  }

  Widget _buildKeypad() {
    return Column(children: [
      Expanded(child: GridView.builder(
        physics: const NeverScrollableScrollPhysics(),
        gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(crossAxisCount: 4, childAspectRatio: 1.25, crossAxisSpacing: 12, mainAxisSpacing: 12),
        itemCount: _keypadGrid.length,
        itemBuilder: (context, index) => _buildButton(_keypadGrid[index]),
      )),
      Padding(padding: const EdgeInsets.only(top: 10), child: SizedBox(width: double.infinity, height: 70, child: JigglyButton(text: '=', backgroundColor: AppTheme.btnEqual, textColor: Colors.black54, onTap: () => _onButtonPressed('=')))),
    ]);
  }

  Widget _buildButton(String val) {
    bool isOp = ['÷', 'x', '-', '+', '%', '(', ')'].contains(val);
    bool isAct = ['C', '⌫']. contains(val);
    return JigglyButton(
      text: val,
      backgroundColor: isOp ? AppTheme.btnOperator : (isAct ? AppTheme. btnAction : AppTheme.btnDefault),
      textColor: isOp ? Colors.white : Colors.black87,
      onTap: () => _onButtonPressed(val),
    );
  }
}

class JigglyButton extends StatefulWidget {
  final String text; final Color backgroundColor; final Color textColor; final VoidCallback onTap;
  const JigglyButton({super.key, required this.text, required this.backgroundColor, required this. textColor, required this.onTap});
  @override State<JigglyButton> createState() => _JigglyButtonState();
}

class _JigglyButtonState extends State<JigglyButton> with SingleTickerProviderStateMixin {
  late AnimationController _c; late Animation<double> _s;
  @override void initState() { super.initState(); _c = AnimationController(vsync: this, duration: const Duration(milliseconds:  100), reverseDuration: const Duration(milliseconds: 300)); _s = Tween<double>(begin: 1.0, end: 0.9).animate(CurvedAnimation(parent: _c, curve:  Curves.easeOut)); }
  @override void dispose() { _c.dispose(); super.dispose(); }
  @override Widget build(BuildContext context) {
    return GestureDetector(
      onTapDown: (_) => _c.forward(), onTapUp: (_) { _c.reverse(); widget.onTap(); }, onTapCancel: () => _c.reverse(),
      child: AnimatedBuilder(animation: _c, builder: (ctx, child) => Transform.scale(scale: _s. value, child: Container(decoration: BoxDecoration(color: widget.backgroundColor, borderRadius: BorderRadius.circular(25), boxShadow: [BoxShadow(color: widget.backgroundColor.withOpacity(0.4), blurRadius: 8, offset: const Offset(0, 4))]), child: Center(child: Text(widget.text, style: TextStyle(fontSize: 22, fontWeight: FontWeight.bold, color: widget.textColor)))))),
    );
  }
}

// --- 5. Strict Parser ---

class ParserException implements Exception {
  final String message;
  ParserException(this.message);
  @override String toString() => message;
}

class Parser {
  final String input; 
  int pos = -1; 
  int ch = -1;
  Parser(this.input);
  
  void nextChar() { ch = (++pos < input.length) ? input.codeUnitAt(pos) : -1; }
  
  bool eat(int charToEat) { 
    while (ch == 32) nextChar(); 
    if (ch == charToEat) { nextChar(); return true; } 
    return false; 
  }
  
  double parse() { 
    nextChar(); 
    double x = parseExpression(); 
    if (pos < input.length) throw ParserException("Unexpected:  ${String.fromCharCode(ch)}"); 
    return x; 
  }

  double parseExpression() { 
    double x = parseTerm(); 
    for (;;) { 
      if (eat(43)) x += parseTerm(); 
      else if (eat(45)) x -= parseTerm(); 
      else return x; 
    } 
  }

  /// FIX 10: Chained Modulo Rejection
  /// 
  /// This method now tracks whether we just processed a modulo operation. 
  /// If we see another '%' immediately after, we throw "Ambiguous". 
  /// This prevents expressions like "10%3%2" from being evaluated. 
  double parseTerm() { 
    double x = parseFactor(); 
    bool lastWasModulo = false;
    
    for (;;) { 
      if (eat(42)) {
        lastWasModulo = false;
        x *= parseFactor(); 
      }
      else if (eat(47)) {
        lastWasModulo = false;
        double divisor = parseFactor();
        if (divisor == 0) throw ParserException("Div by Zero");
        x /= divisor;
      }
      else if (eat(37)) {
        if (lastWasModulo) {
          throw ParserException("Ambiguous");
        }
        double divisor = parseFactor();
        if (x % 1 != 0 || divisor % 1 != 0) throw ParserException("Int Mod Only");
        if (divisor == 0) throw ParserException("Div by Zero");
        x %= divisor;
        lastWasModulo = true;
      }
      else {
        return x; 
      }
    } 
  }
  
  double parseFactor() {
    if (eat(43)) throw ParserException("Unexpected: +");
    if (eat(45)) return -parseFactor();
    
    double x; int startPos = pos;
    
    if (eat(40)) { 
      x = parseExpression(); 
      if (! eat(41)) throw ParserException("Missing ')'"); 
    } 
    else if ((ch >= 48 && ch <= 57) || ch == 46) {
      while ((ch >= 48 && ch <= 57) || ch == 46) nextChar();
      String sub = input.substring(startPos, pos);
      // 3.  Reject Standalone Decimal
      if (sub == '. ') throw ParserException("Invalid Number"); 
      x = double.parse(sub);
    } else { 
      // 2. Strict EOF / Unexpected
      if (ch == -1) throw ParserException("Incomplete");
      throw ParserException("Unexpected: ${String.fromCharCode(ch)}"); 
    }
    return x;
  }
}
