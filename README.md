# tai42-webhook-verifier-stripe

Stripe webhook verifier plugin for the TAI42 platform.

Registers a `stripe` `WebhookVerifier` that validates Stripe's
`Stripe-Signature` header against the endpoint signing secret, so signed
Stripe events can answer external `ask_user` questions.
