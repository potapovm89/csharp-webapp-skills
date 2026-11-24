# Merge Request Description Template

Every MR must provide this structured summary:

```markdown
## Что сделано
- Краткий список изменений по существу

## Задача
<номер и ссылка на задачу в трекере>

## Как проверить
- Шаги локальной проверки / какие тесты покрывают изменения

## Изменения вне кода
- Миграции БД: есть/нет (какие именно)
- Изменения контрактов: есть/нет (обратная совместимость сохранена?)
- Изменения настроек: есть/нет (какие новые параметры добавить при деплое)
- Обновлена документация в docs/: перечень обновлённых файлов
- Поставка/надёжность/восстановление: затронутые gates, SLI/SLO или RPO/RTO и ссылки на соответствующие проверки; если неприменимо — краткая причина
```

## Checklist before submitting MR:
- [ ] Approvals match the review policy: security, migration, delivery-gate and WorkerHost-clustering changes include the designated reviewer (`git-workflow`, branching.md).
- [ ] For application code changes, `dotnet build --no-incremental` succeeds with 0 warnings and 0 errors (an incremental build may skip analyzers in unchanged projects).
- [ ] Relevant tests and required CI gates pass; actual commands/results and any unavailable checks are recorded. Documentation-only changes have appropriate validation.
- [ ] XML-documentation completed on all public interfaces, models, and controllers.
- [ ] Affected behavior/contracts/settings/schema/permissions are documented in `docs/`, or no documentation impact is explained.
- [ ] No secrets leaked into any tracked files.
- [ ] If delivery, critical journeys or persistent-state recovery changed, the applicable gate/trust, load/failure or restore evidence and documentation are linked (`delivery-and-supply-chain`, `reliability-and-performance`, `data-recovery`). An unperformed required check is reported, not marked passed.
