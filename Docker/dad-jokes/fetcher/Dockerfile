FROM alpine:latest

# 1. Install dependencies (as root)
# 1. FIXED: Use apk for Alpine, not apt-get
RUN apk add --no-cache curl ca-certificates
# RUN apt-get update && \
#     apt-get install -y --no-install-recommends curl ca-certificates && \
#     rm -rf /var/lib/apt/lists/*

# 2. Create a user with UID 101 to match the Nginx Unprivileged image
# -D: Don't assign a password
# -u: Force specific User ID
RUN adduser -D -u 101 appuser

# 3. Set up the app directory
WORKDIR /app
COPY updater .
RUN chmod +x updater

# 4. Create the data directory and assign permissions to our user
RUN mkdir /data && chown 101:101 /data

# 5. Switch to the non-privileged user
USER 101

ENTRYPOINT ["./updater"]