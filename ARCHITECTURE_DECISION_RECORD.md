# Reducer-Based State + Plugin Slices (подход ProseMirror)

**Как заменяется pub-sub**  

| Было (pub-sub) | Стало (reducer-based) |
| ------------------------------------------------ | --------------------------------------------------------------- |
| pluginSubscriptions.emit([cursor], ON_CURSOR) | dispatch({ type: 'CURSOR_SET', payload: cursor }) |
| subscribeEvent(name, ON_CURSOR, handler) | reducer(state, tr) { if (tr.action.type === 'CURSOR_SET') ... } |
| subscribeEvent(name, ON_SELECTED_RANGE, handler) | reducer(state, tr) { if (tr.action.type === 'RANGES_SET') ... } |
| Реактивная логика в обработчиках событий | appendTransaction() — генерирует повторные транзакции |
| setState({ isActive: true }) внутри плагина | Возвращается новый объект из reducer |
  
**Поток данных**  
```
Пользователь -> dispatch(action)
               ↓
         Transaction()
               ↓
    ┌─── СТАДИЯ 1: filterTransaction ───┐
    │    (блокировка транзакции)        │
    └───────────────────────────────────┘
               ↓
    ┌─── СТАДИЯ 2: reducer ─────────────┐
    │    CorePlugin     <- cursor, ranges│
    │    PluginFormulas <- edit state    │
    │    PluginCellStyling <- style state│
    └───────────────────────────────────┘
               ↓
    ┌─── СТАДИЯ 3: appendTransaction ───┐
    │    Побочные транзакции            │
    │    (напр. RANGES_STYLES_SET)      │
    └───────────────────────────────────┘
               ↓
         _triggerUpdate() -> React re-render
```

**Итоговая схема потока данных (полная)**  
```
┌────────────────────────────────────────────────────────────────────┐
│                        AdapterSpreadSheet                          │
│                                                                    │
│  render()           <- читает _coreState (из SpreadsheetAdapter)    │
│  dispatch(action)   -> SpreadsheetAdapter._applyTransaction()       │
│  _triggerUpdate()   <- вызывается после каждого dispatch            │
│                                                                    │
│  dataMatrix         <- мутабельная ссылка (не в reducer-state)      │
│  styleManager       <- мутабельная ссылка (не в reducer-state)      │
└──────────────────────────────┬─────────────────────────────────────┘
                               │ dispatch
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│                      SpreadsheetAdapter                            │
│                                                                    │
│  state: {                                                          │
│    core:         CoreState          <- cursor, ranges, meta...      │
│    PluginFormulas: PluginFormulasState                             │
│    PluginCellStyling: PluginCellStylingState                       │
│  }                                                                 │
│                                                                    │
│  _applyTransaction(tr):                                            │
│    1. filterTransaction  -> может заблокировать                     │
│    2. reducer × N        -> новый иммутабельный state               │
│    3. appendTransaction  -> генерирует доп. транзакции              │
└────────────────────────────────────────────────────────────────────┘
                               │ getState()
                    ┌──────────┼──────────┐
                    ▼          ▼          ▼
              CorePlugin  CellEdit  CellStyling
              reducer()   reducer() reducer()
                    │          │          │
                    │ appendTransaction   │
                    └──────────┼──────────┘
                               │ SpreadsheetAction[]
                               ▼
                      повторный dispatch
                      (skipHistory=true)

```
**Пример использования**  
```typescript
// App.tsx

import React from 'react';
import AdapterSpreadSheet from './AdapterSpreadSheet';
import { CanvasTableAdapter } from './TableAdapters/CanvasTableAdapter';
import { PluginFormulas } from './plugins/PluginFormulas';
import { PluginCellStyling } from './plugins/PluginCellStyling';
import { PluginsMap } from './AdapterSpreadSheet/types';

const plugins: PluginsMap = {
    cellEdit: {
        plugin: new PluginFormulas(),
        rootId: 'formula-bar-portal',  // Portal для строки формул
    },
    cellStyling: {
        plugin: new PluginCellStyling(),
        rootId: 'toolbar-portal',      // Portal для панели инструментов
    },
};

export function App() {
    return (
        <>
            {/* Контейнеры для Portal UI плагинов */}
            <div id="formula-bar-portal" style={{ display: 'flex', alignItems: 'center' }} />
            <div id="toolbar-portal" style={{ display: 'flex', flexWrap: 'wrap' }} />

            {/* Таблица */}
            <AdapterSpreadSheet
                plugins={plugins}
                tableAdapter={CanvasTableAdapter}
                style={{ width: '100%', height: 'calc(100vh - 80px)' }}
            />
        </>
    );
}

```
**Пример создания нового плагина — PluginValidation**  
Показывает использование всех трёх стадий:
```typescript
import React from 'react';
import { Cell } from '../../AdapterSpreadSheet/models';
import { CellMeta, Plugin } from '../../AdapterSpreadSheet/plugin/Plugin';
import { SpreadsheetAction } from '../../AdapterSpreadSheet/plugin/SpreadsheetAction';
import { Transaction } from '../../AdapterSpreadSheet/plugin/Transaction';
import { SparseMatrixHelper } from '../../AdapterSpreadSheet/utils/SparseMatrixHelper/SparseMatrixHelper';

export type ValidationRule = {
    rowIndex: number;
    columnIndex: number;
    validate: (value: any) => boolean;
    message: string;
};

export interface PluginValidationState {
    rules: ValidationRule[];
    violations: Map<string, string>; // "row:col" -> message
}

export const PLUGIN_VALIDATION_KEY = 'PluginValidation';

export class PluginValidation extends Plugin<PluginValidationState> {
    readonly key = PLUGIN_VALIDATION_KEY;

    readonly initialState: PluginValidationState = {
        rules:      [],
        violations: new Map(),
    };

    // ─── СТАДИЯ 1: блокируем невалидные записи ────────────────────────────────

    filterTransaction(tr: Transaction, state: PluginValidationState): boolean {
        if (tr.action.type !== 'CELLS_SET') return true;

        const { data } = tr.action.payload;

        for (const rule of state.rules) {
            const row = data.get(rule.rowIndex);
            if (!row?.has(rule.columnIndex)) continue;

            const newValue = row.get(rule.columnIndex)?.data;
            if (!rule.validate(newValue)) {
                // Блокируем транзакцию — невалидное значение
                console.warn(
                    `[PluginValidation] Blocked write to R${rule.rowIndex}C${rule.columnIndex}: ${rule.message}`,
                );
                return false;
            }
        }

        return true;
    }

    // ─── СТАДИЯ 2: обновляем список нарушений ─────────────────────────────────

    reducer(state: PluginValidationState, tr: Transaction): PluginValidationState {
        switch (tr.action.type) {
            case 'CELLS_SET': {
                // Пересчитываем нарушения для изменённых ячеек
                const newViolations = new Map(state.violations);

                tr.action.payload.data.forEach((row, rowIndex) => {
                    row.forEach((cell, columnIndex) => {
                        const key = `${rowIndex}:${columnIndex}`;
                        const rule = state.rules.find(
                            (r) => r.rowIndex === rowIndex && r.columnIndex === columnIndex,
                        );
                        if (!rule) {
                            newViolations.delete(key);
                            return;
                        }
                        if (!rule.validate(cell?.data)) {
                            newViolations.set(key, rule.message);
                        } else {
                            newViolations.delete(key);
                        }
                    });
                });

                if (newViolations.size === state.violations.size &&
                    [...newViolations].every(([k, v]) => state.violations.get(k) === v)) {
                    return state;
                }

                return { ...state, violations: newViolations };
            }

            case `${PLUGIN_VALIDATION_KEY}/ADD_RULE`:
                return { ...state, rules: [...state.rules, action.payload] };

            case `${PLUGIN_VALIDATION_KEY}/REMOVE_RULE`: {
                const { rowIndex, columnIndex } = action.payload;
                return {
                    ...state,
                    rules: state.rules.filter(
                        (r) => !(r.rowIndex === rowIndex && r.columnIndex === columnIndex),
                    ),
                };
            }

            case 'CLEAR':
                return { ...state, violations: new Map() };

            default:
                return state;
        }
    }

    // ─── СТАДИЯ 3: при появлении нарушений — помечаем ячейки ─────────────────

    appendTransaction(
        tr: Transaction,
        prevState: PluginValidationState,
        nextState: PluginValidationState,
    ): SpreadsheetAction | null {
        if (prevState.violations === nextState.violations) return null;

        // Генерируем стили для ячеек с ошибками через RANGE_STYLE_SET
        // (в реальном коде это могло бы обновлять styleManager)
        // Здесь просто эмитируем кастомный экшен для внешних слушателей
        return {
            type: `${PLUGIN_VALIDATION_KEY}/VIOLATIONS_UPDATED`,
            payload: Object.fromEntries(nextState.violations),
        };
    }

    // ─── getCellStyle: подсвечиваем невалидные ячейки ─────────────────────────

    getCellStyle(state: PluginValidationState, cell: CellMeta): React.CSSProperties {
        const key = `${cell.rowIndex}:${cell.columnIndex}`;
        if (!state.violations.has(key)) return {};

        return {
            outline:         '2px solid #FF4444',
            outlineOffset:   '-1px',
            backgroundColor: 'rgba(255, 68, 68, 0.05)',
        };
    }

    // ─── Public API: добавление/удаление правил ───────────────────────────────

    addRule(rule: ValidationRule): void {
        this.context.dispatch({
            type:    `${PLUGIN_VALIDATION_KEY}/ADD_RULE`,
            payload: rule,
        });
    }

    removeRule(rowIndex: number, columnIndex: number): void {
        this.context.dispatch({
            type:    `${PLUGIN_VALIDATION_KEY}/REMOVE_RULE`,
            payload: { rowIndex, columnIndex },
        });
    }

    getViolations(): Map<string, string> {
        return this.getState().violations;
    }

    hasViolation(rowIndex: number, columnIndex: number): boolean {
        return this.getState().violations.has(`${rowIndex}:${columnIndex}`);
    }
}

```

**Руководство по миграции старых плагинов**  
```typescript
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// БЫЛО: AdapterSpreadSheetPlugin (pub-sub)
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

class MyOldPlugin extends AdapterSpreadSheetPlugin<Options, State> {
    constructor(args) {
        super(args);
        this.state = { counter: 0 };
    }

    subscribesEvents(pluginName, subscribeEvent) {
        subscribeEvent(pluginName, ON_CURSOR, this.onCursor);
        subscribeEvent(pluginName, ON_SELECTED_RANGE, this.onSelectedRange);
        subscribeEvent(pluginName, CLEAR_EVENT, this.onClear);
    }

    onCursor = (cursor) => {
        this.setState((s) => ({ ...s, counter: s.counter + 1 }));
    };

    onSelectedRange = (ranges) => {
        // ...
        this.tableAdapter.setSelectedRangesStyles(styles);
    };

    onClear = () => {
        this.setState(() => ({ counter: 0 }));
    };

    render() {
        return <div>{this.state.counter}</div>;
    }
}

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// СТАЛО: Plugin<S> (reducer-based)
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

interface MyPluginState {
    counter: number;
}

class MyNewPlugin extends Plugin<MyPluginState> {
    readonly key = 'MyPlugin';
    readonly initialState = { counter: 0 };

    // ① subscribesEvents -> reducer
    reducer(state: MyPluginState, tr: Transaction): MyPluginState {
        switch (tr.action.type) {
            case 'CURSOR_SET':          // вместо subscribeEvent(ON_CURSOR)
                return { ...state, counter: state.counter + 1 };

            case 'CLEAR':               // вместо subscribeEvent(CLEAR_EVENT)
                return { counter: 0 };

            default:
                return state;
        }
    }

    // ② setState + tableAdapter.set*() -> appendTransaction
    appendTransaction(tr, prevState, nextState) {
        if (tr.action.type === 'RANGES_SET') {
            const styles = this._computeStyles(tr.action.payload);
            return { type: 'RANGES_STYLES_SET', payload: styles };
        }
        return null;
    }

    private _computeStyles(ranges: Range[]): Record<string, IRangeStyles> {
        // ...
        return {};
    }

    // ③ render — читаем через getState() вместо this.state
    render() {
        const state = this.getState(); // <- актуальный стейт
        return <div>{state.counter}</div>;
    }
}

```
  
**Сводная таблица замен**  

| Старый API | Новый API | Где |
| --------------------------------------------- | --------------------------------------------------------------- | ------------------ |
| subscribeEvent(name, ON_CURSOR, fn) | reducer: case 'CURSOR_SET' | Plugin |
| subscribeEvent(name, ON_SELECTED_RANGE, fn) | reducer: case 'RANGES_SET' | Plugin |
| subscribeEvent(name, CLEAR_EVENT, fn) | reducer: case 'CLEAR' | Plugin |
| subscribeEvent(name, ON_CELL_EDIT_START, fn) | reducer: case 'CELL_EDIT_START' | Plugin |
| subscribeEvent(name, ON_CELL_EDIT_INPUT, fn) | reducer: case 'CELL_EDIT_INPUT' | Plugin |
| subscribeEvent(name, ON_CELL_EDIT_END, fn) | appendTransaction: if CELL_EDIT_END | Plugin |
| this.setState(fn) | Возврат нового объекта из reducer | Plugin |
| this.tableAdapter.setSelectedRangesStyles(s) | appendTransaction -> RANGES_STYLES_SET | Plugin |
| this.tableAdapter.setCells(data) | appendTransaction -> CELLS_SET | Plugin |
| this.tableAdapter.clearSelectedRangesStyles() | appendTransaction -> RANGES_STYLES_SET | Plugin |
| pluginSubscriptions.emit([data], event) | dispatch({ type: event, payload: data }) | AdapterSpreadSheet |
| this.tableAdapter._triggerUpdate() | this.context.dispatch(any) | Plugin |
| this.state.xxx в render() | this.getState().xxx | Plugin |
| this.tableAdapter.getCursor() | this.context.getPluginState<PluginCursorCellState>(PLUGIN_CURSOR_CELL_KEY)?.cursor | Plugin |
  
**Что изменилось в типах**  
**§ 6 — Система плагинов**  

| Было | Стало |
| --------------------------------------------------------- | ------------------------------------------- |
| AdapterSpreadSheetPropsPlugin<O> с component: Constructor | PluginEntry<S> с plugin: Plugin<S> |
| IAdapterSpreadSheetPluginsMap | PluginsMap = Record<string, PluginEntry> |
| AdapterSpreadSheetPluginArgs | Инжекция через Plugin._inject(ctx, options) |
  
```
AdapterSpreadSheetPropsPlugin и IAdapterSpreadSheetPluginsMap помечены @deprecated — оставлены для постепенной миграции.

```
**§ 7 — Состояние таблицы**  
```
AdapterSpreadSheetState сохранён как тип для TypeScript-совместимости с Component<Props, State>. Фактически хранится в SpreadsheetAdapter (CorePlugin), в React state только поле lastUpdate.

```
**§ 11 — IAdapter — новые методы**  
```
dispatch:      (action: SpreadsheetAction) => void;   // единственная точка записи
undo:          () => void;
redo:          () => void;
getPluginState:<S>(key: string) => S | undefined;     // читать стейт любого плагина
snapshotState: () => PluginStatesMap;
restoreState:  (snap: PluginStatesMap) => void;

```
**§ 12 — Adapter — убраны зависимости от pub-sub**  
* Удалены все упоминания PluginSubscriptions  
* _triggerUpdate вместо forceUpdate (единый механизм перерисовки)  
* setState/update/promiseState больше не принимают бизнес-стейт  
**§ 13 — Props**  
```
// Было:
plugins?: Partial<IAdapterSpreadSheetPluginsMap>

// Стало:
plugins?: PluginsMap  // Record<string, PluginEntry<S>>

```
