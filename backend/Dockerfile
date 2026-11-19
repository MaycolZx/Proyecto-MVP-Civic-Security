# Imagen multi-etapa para ejecutar Civic Security como `mix release` en producción.
# Mismas versiones de Elixir y OTP que CI (.github/workflows/ci.yml).
# Compatible con x86_64 (EC2 t3.*) y ARM64 (EC2 t4g.*).

FROM elixir:1.17.3-otp-27-slim AS builder

RUN apt-get update -y && \
    apt-get install -y --no-install-recommends build-essential git ca-certificates && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

RUN mix local.hex --force && \
    mix local.rebar --force

ENV MIX_ENV=prod

COPY mix.exs mix.lock ./
RUN mix deps.get --only prod

RUN mkdir config
COPY config/config.exs config/prod.exs config/
RUN mix deps.compile

COPY priv priv
COPY lib lib
RUN mix compile

# runtime.exs se evalúa al arrancar, no al compilar
COPY config/runtime.exs config/
RUN mix release

# --- Imagen final mínima, sin herramientas de compilación ni usuario root ---
FROM debian:bookworm-slim AS runner

RUN apt-get update -y && \
    apt-get install -y --no-install-recommends \
      libstdc++6 \
      openssl \
      libncurses6 \
      locales \
      ca-certificates \
      curl && \
    rm -rf /var/lib/apt/lists/* && \
    sed -i '/en_US.UTF-8/s/^# //g' /etc/locale.gen && \
    locale-gen

ENV LANG=en_US.UTF-8 \
    LANGUAGE=en_US:en \
    LC_ALL=en_US.UTF-8 \
    MIX_ENV=prod \
    PHX_SERVER=true

# Usuario sin privilegios y sin shell interactiva (ver docs/despliegue.md)
RUN groupadd --gid 1000 civic && \
    useradd --uid 1000 --gid civic --no-create-home --shell /usr/sbin/nologin civic && \
    mkdir -p /app /var/lib/civic/uploads && \
    chown -R civic:civic /app /var/lib/civic && \
    chmod 0700 /var/lib/civic/uploads

WORKDIR /app

COPY --from=builder --chown=civic:civic /app/_build/prod/rel/civic ./

USER civic

EXPOSE 4000

# 127.0.0.1 está excluido de force_ssl en config/prod.exs
HEALTHCHECK --interval=15s --timeout=5s --start-period=20s --retries=3 \
  CMD curl -fsS http://127.0.0.1:4000/health || exit 1

CMD ["/app/bin/civic", "start"]
