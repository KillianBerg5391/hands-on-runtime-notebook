# Protected Data Export: 3 Gates for Consent, Session Verification, and Risk Review

Short answer: authorize a protected data export only after three independent gates pass: current consent, recent session verification, and a recorded risk decision.

That order matters in a customer-support system migrating away from managed authentication. A password reset proves control of a recovery channel; it should not silently become approval to export an account history. Keep reset, sign-in, export authorization, and export delivery as separate state transitions, tied together by one stable internal subject ID rather than a provider-specific user ID.

No shortcuts.

## What constraint changes the forgot-password design?

The awkward constraint is migration. Old and new authentication sessions may coexist while support agents still need to answer export-status questions. If export authorization depends directly on a managed provider's session shape, migration code leaks into the most sensitive workflow and every downstream check gains a second branch.

Put a narrow identity boundary in front of the flow instead. It accepts a provider assertion, resolves it to an internal subject, and returns the authentication time and methods that the application understands. The export service never sees a provider token. This also keeps the support role out of authorization: an agent can explain status or restart a failed identity-verification process, but cannot mark consent as present or approve risk on the customer's behalf.

OWASP recommends reauthentication after high-risk events, including account recovery and password resets, and for critical actions. It also recommends rotating or invalidating sessions after reauthentication. That makes the migration rule fairly crisp: a reset can revoke older sessions, while an export requires a newly verified session issued under the current trust policy. Don't infer freshness from the token's mere existence.

There is a DX payoff — a small one, but real. Authentication adapters change during migration; the export decision function does not. I would benchmark the adapters on time-to-first verified subject, number of required configuration fields, and failure-path parity before moving traffic. Those are proposed acceptance measures, not claims about any provider.

## How should a protected data export combine consent, session verification, and risk review?

Treat the three gates as inputs to one fail-closed decision, not as middleware that happens to run in a convenient order. Consent answers whether this subject approved this export request and scope. Session verification answers whether the present actor recently proved control under the required authentication policy. Risk review answers whether context around the action permits automatic release, requires manual review, or requires another verification step.

Bind all four.

The distinction prevents a nasty class of support mistakes. Imagine request `exp_7F3A`: the customer resets a password, signs in, approves an export of ticket messages, and then changes the destination. Consent for the earlier scope cannot authorize the changed package. A risk result created before that change should not drift forward either. Bind every result to the internal subject, export ID, normalized scope digest, and policy version. If any binding differs, return a denial reason such as `CONSENT_SCOPE_MISMATCH` and create a fresh decision. This error name is an application design choice, useful because it is stable, searchable, and safe to expose only in internal logs.

The risk engine should return a decision, not a magic score that every caller interprets differently. `allow`, `step_up`, `manual_review`, and `deny` are enough for the surrounding workflow to remain explicit. I'm not sure a universal freshness interval exists; the cited guidance calls for context-aware reauthentication rather than one fixed duration. Resolve that uncertainty with a documented policy based on export sensitivity and authentication method, then test the boundary values. Your mileage may vary.

Audit each gate separately, but keep secrets out of the event. Record subject ID, request ID, scope digest, policy version, decision, reason code, and timestamps. Do not log passwords, recovery answers, raw session tokens, or the export itself. The audit trail should explain why release was allowed without becoming a second copy of protected customer data.

## The smallest working decision core

This is the part worth keeping boring. The function below has no SDK, network client, or provider configuration. Its callers load verified records from their own stores, and its output can drive a queue or a manual-review state machine.

```ts
type RiskDecision = "allow" | "step_up" | "manual_review" | "deny";

type ExportContext = {
  exportId: string;
  subjectId: string;
  scopeDigest: string;
  policyVersion: string;
  nowMs: number;
};

type ConsentRecord = {
  subjectId: string;
  scopeDigest: string;
  policyVersion: string;
  approvedAtMs: number;
  revokedAtMs?: number;
};

type VerifiedSession = {
  subjectId: string;
  verifiedAtMs: number;
  authenticationMethods: string[];
  invalidatedAtMs?: number;
};

type RiskReview = {
  exportId: string;
  subjectId: string;
  scopeDigest: string;
  policyVersion: string;
  decision: RiskDecision;
  reviewedAtMs: number;
};

type ExportDecision =
  | { release: true; auditReason: "ALL_GATES_PASSED" }
  | {
      release: false;
      auditReason:
        | "CONSENT_INVALID"
        | "SESSION_INVALID"
        | "SESSION_TOO_OLD"
        | "RISK_REVIEW_STALE"
        | "RISK_STEP_UP"
        | "RISK_MANUAL_REVIEW"
        | "RISK_DENIED";
    };

function authorizeExport(
  context: ExportContext,
  consent: ConsentRecord,
  session: VerifiedSession,
  risk: RiskReview,
  maximumSessionAgeMs: number,
): ExportDecision {
  const consentMatches =
    consent.subjectId === context.subjectId &&
    consent.scopeDigest === context.scopeDigest &&
    consent.policyVersion === context.policyVersion &&
    consent.revokedAtMs === undefined;

  if (!consentMatches) return { release: false, auditReason: "CONSENT_INVALID" };

  if (session.subjectId !== context.subjectId || session.invalidatedAtMs !== undefined) {
    return { release: false, auditReason: "SESSION_INVALID" };
  }

  if (context.nowMs - session.verifiedAtMs > maximumSessionAgeMs) {
    return { release: false, auditReason: "SESSION_TOO_OLD" };
  }

  const riskMatches =
    risk.exportId === context.exportId &&
    risk.subjectId === context.subjectId &&
    risk.scopeDigest === context.scopeDigest &&
    risk.policyVersion === context.policyVersion;

  if (!riskMatches) return { release: false, auditReason: "RISK_REVIEW_STALE" };
  if (risk.decision === "step_up") return { release: false, auditReason: "RISK_STEP_UP" };
  if (risk.decision === "manual_review") {
    return { release: false, auditReason: "RISK_MANUAL_REVIEW" };
  }
  if (risk.decision === "deny") return { release: false, auditReason: "RISK_DENIED" };

  return { release: true, auditReason: "ALL_GATES_PASSED" };
}
```

Notice what is absent: the function does not trust a support-agent flag, mutate consent, refresh a session, or reinterpret risk. Each of those actions belongs to a separately authorized command. That separation makes retries predictable. A queue worker can call the function again with the same immutable inputs and get the same answer; a changed scope or policy produces new inputs and a new audit event.

The catch is extra state. A tiny application with no sensitive exports may reasonably keep authorization closer to its request handler. This design is not suitable when the team cannot operate durable consent and review records; in that case, disable self-service export and use a documented manual process until those controls exist. Do not preserve the appearance of automation with checks that cannot be audited.

## Test failure paths before release

The happy path is one test. The useful suite is a matrix: revoked consent, changed scope, old session, session invalidated after password reset, mismatched subject, stale policy version, step-up decision, manual review, denial, and repeated execution with identical inputs. Assert both the release boolean and the audit reason. Then test that logs omit every raw credential and export field.

Race conditions deserve their own test. Pause the release worker after authorization, revoke consent, then resume it. If release still occurs, the decision was made too early. Either place the final check in the same transaction as the transition to an immutable release job, or make the delivery worker verify a short-lived, single-use authorization artifact bound to the exact package digest. The mechanism can differ. The invariant cannot: revocation before the release boundary must win.

Test that race.

Also exercise enumeration resistance around forgot-password entry points. OWASP advises returning a consistent message and taking a consistent amount of time for existing and nonexistent accounts. The support UI should inherit that posture; an agent-facing lookup must not turn a public recovery request into an account-existence oracle.

## What I would change at scale

At higher volume, I would split decision production from export generation, keep policy versions immutable, and measure queue age, step-up frequency, manual-review age, denial reasons, and releases canceled before delivery. I would not start with a rules language. Plain typed decisions are easier to inspect, migrate, and replay; introduce a policy engine only when multiple teams truly need independent rule changes and can test those changes against recorded, de-identified cases.

This architecture has costs: more records, more transitions, and a support playbook that must explain `step_up` versus `manual_review`. Stick with a simpler synchronous design when export scope is narrow, traffic is low, and one team owns the full boundary. Choose the state machine when provider migration, delayed generation, or independent review makes a single request lifecycle dishonest.

The decision rule remains vendor-neutral: release only the exact package that the subject approved, from a recently verified session, under a risk decision bound to the same request and policy. Everything else waits.

## References

- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
