# Builds the custom Keycloak image used by helm/charts/keycloak/values.yaml
# (vnair5/pf-keycloak) — bakes the personal-finance Keycloakify theme jar
# into the stock quay.io/keycloak/keycloak image.
#
# Build context must be the REPO ROOT, not helm/charts/keycloak/, since the
# COPY below reaches into keycloak-theme/dist_keycloak/:
#   docker buildx build --platform linux/amd64,linux/arm64 \
#     -f helm/charts/keycloak/Dockerfile -t vnair5/pf-keycloak:latest --push .
#
# Uses the "all other versions" theme jar variant (NOT the kc-22-to-25
# variant used by the local docker-compose stack), because this chart's
# pinned Keycloak version (see values.yaml image.tag, currently 26.2.4) is
# outside the 22-25 range that the other jar targets. If the pinned version
# in values.yaml ever changes into the 22-25 range, switch back to
# keycloak-theme-for-kc-22-to-25.jar instead.
FROM quay.io/keycloak/keycloak:26.2.4

COPY keycloak-theme/dist_keycloak/keycloak-theme-for-kc-all-other-versions.jar /opt/keycloak/providers/keycloak-theme-for-kc-all-other-versions.jar
