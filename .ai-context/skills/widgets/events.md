# Events

- **Widget ID:** `com.mendix.widget.web.events.Events`
- **Type:** PLUGGABLEWIDGET
- **Version:** 1.3.1

## MDL Example

```sql
PLUGGABLEWIDGET 'com.mendix.widget.web.events.Events' widget1
```

## Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `onComponentLoad` | action |  |  |  |
| `componentLoadDelayParameterType` | enumeration | Yes | number |  |
| `componentLoadDelay` | integer | Yes | 0 | Timer delay to first action execution. Value is in milliseconds. If set to 0,... |
| `componentLoadDelayExpression` | expression | Yes |  | Timer delay to first action execution. Value is in milliseconds. If set to 0,... |
| `componentLoadRepeat` | boolean | Yes | false |  |
| `componentLoadRepeatIntervalParameterType` | enumeration | Yes | number |  |
| `componentLoadRepeatInterval` | integer | Yes | 30000 | Interval between repeat action execution. Value is in milliseconds. |
| `componentLoadRepeatIntervalExpression` | expression |  |  | Interval between repeat action execution. Value is in milliseconds. |
| `onEventChangeAttribute` | attribute |  |  |  |
| `onEventChange` | action |  |  |  |
| `onEventChangeDelayParameterType` | enumeration | Yes | number |  |
| `onEventChangeDelay` | integer | Yes | 0 | Timer delay to first action execution. Value is in milliseconds. If set to 0,... |
| `onEventChangeDelayExpression` | expression | Yes |  | Timer delay to first action execution. Value is in milliseconds. If set to 0,... |

